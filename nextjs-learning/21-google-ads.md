# 第 21 章：Google AdSense 與 Google Ads 整合

## 21.1 Google AdSense 簡介

Google AdSense 是 Google 提供的廣告服務，可讓網站擁有者在其網站上顯示相關廣告並賺取收入。在 Next.js 應用程式中整合 AdSense 需要特別注意 SSR（伺服器端渲染）和 SSG（靜態網站生成）的環境。

## 21.2 環境設定

### 21.2.1 取得 AdSense 帳號

1. 前往 [Google AdSense](https://www.google.com/adsense/)
2. 註冊並驗證您的網站
3. 取得您的發布者 ID（格式：ca-pub-xxxxxxxxxxxxxxxx）

### 21.2.2 安裝必要套件

```bash
npm install next-google-adsense
# 或
yarn add next-google-adsense
```

## 21.3 基本整合

### 21.3.1 在 _app.tsx 中加入 AdSense 腳本

使用 Next.js 的 `Script` 組件可以優化廣告腳本的載入：

```tsx
// pages/_app.tsx
import type { AppProps } from 'next/app'
import Script from 'next/script'

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <>
      {/* Google AdSense 腳本 */}
      <Script
        id="google-adsense"
        async
        src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-xxxxxxxxxxxxxxxx"
        crossOrigin="anonymous"
        strategy="afterInteractive"
      />

      <Component {...pageProps} />
    </>
  )
}

export default MyApp
```

### 21.3.2 Script 組件的 strategy 屬性說明

- **beforeInteractive**：在頁面互動前載入（適用於關鍵腳本）
- **afterInteractive**：在頁面互動後載入（推薦用於 AdSense）
- **lazyOnload**：在瀏覽器閒置時載入（延遲載入）

## 21.4 建立可重用的廣告組件

### 21.4.1 基本廣告組件

```tsx
// components/AdSense/AdUnit.tsx
import { useEffect, useRef } from 'react'

interface AdUnitProps {
  adSlot: string
  adFormat?: string
  fullWidthResponsive?: boolean
  style?: React.CSSProperties
  className?: string
}

declare global {
  interface Window {
    adsbygoogle: any[]
  }
}

export default function AdUnit({
  adSlot,
  adFormat = 'auto',
  fullWidthResponsive = true,
  style = { display: 'block' },
  className = ''
}: AdUnitProps) {
  const adRef = useRef<HTMLModElement>(null)

  useEffect(() => {
    try {
      // 確保廣告腳本已載入
      if (typeof window !== 'undefined' && window.adsbygoogle) {
        // 推送廣告到隊列
        ;(window.adsbygoogle = window.adsbygoogle || []).push({})
      }
    } catch (error) {
      console.error('AdSense error:', error)
    }
  }, [])

  return (
    <ins
      ref={adRef}
      className={`adsbygoogle ${className}`}
      style={style}
      data-ad-client="ca-pub-xxxxxxxxxxxxxxxx"
      data-ad-slot={adSlot}
      data-ad-format={adFormat}
      data-full-width-responsive={fullWidthResponsive.toString()}
    />
  )
}
```

### 21.4.2 響應式廣告組件

```tsx
// components/AdSense/ResponsiveAd.tsx
import AdUnit from './AdUnit'

interface ResponsiveAdProps {
  adSlot: string
  className?: string
}

export default function ResponsiveAd({ adSlot, className }: ResponsiveAdProps) {
  return (
    <div className={`ad-container ${className || ''}`}>
      <AdUnit
        adSlot={adSlot}
        adFormat="auto"
        fullWidthResponsive={true}
        style={{ display: 'block' }}
      />
    </div>
  )
}
```

### 21.4.3 固定尺寸廣告組件

```tsx
// components/AdSense/FixedAd.tsx
import AdUnit from './AdUnit'

interface FixedAdProps {
  adSlot: string
  width: number
  height: number
  className?: string
}

export default function FixedAd({
  adSlot,
  width,
  height,
  className
}: FixedAdProps) {
  return (
    <div className={`ad-container ${className || ''}`}>
      <AdUnit
        adSlot={adSlot}
        adFormat="fixed"
        fullWidthResponsive={false}
        style={{
          display: 'inline-block',
          width: `${width}px`,
          height: `${height}px`
        }}
      />
    </div>
  )
}
```

## 21.5 廣告單元放置策略

### 21.5.1 頁面頂部廣告

```tsx
// pages/blog/[slug].tsx
import ResponsiveAd from '@/components/AdSense/ResponsiveAd'

export default function BlogPost({ post }: { post: any }) {
  return (
    <article>
      {/* 頁面頂部橫幅廣告 */}
      <ResponsiveAd
        adSlot="1234567890"
        className="mb-8"
      />

      <h1>{post.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: post.content }} />
    </article>
  )
}
```

### 21.5.2 文章內廣告

```tsx
// components/Article/ArticleContent.tsx
import ResponsiveAd from '@/components/AdSense/ResponsiveAd'

interface ArticleContentProps {
  content: string
}

export default function ArticleContent({ content }: ArticleContentProps) {
  // 將內容分段
  const paragraphs = content.split('\n\n')
  const midPoint = Math.floor(paragraphs.length / 2)

  return (
    <div className="article-content">
      {/* 前半部分內容 */}
      {paragraphs.slice(0, midPoint).map((para, idx) => (
        <p key={idx}>{para}</p>
      ))}

      {/* 文章中間插入廣告 */}
      <ResponsiveAd
        adSlot="9876543210"
        className="my-8"
      />

      {/* 後半部分內容 */}
      {paragraphs.slice(midPoint).map((para, idx) => (
        <p key={idx}>{para}</p>
      ))}
    </div>
  )
}
```

### 21.5.3 側邊欄廣告

```tsx
// components/Layout/Sidebar.tsx
import FixedAd from '@/components/AdSense/FixedAd'

export default function Sidebar() {
  return (
    <aside className="sidebar">
      <div className="sticky top-4">
        {/* 側邊欄固定尺寸廣告 */}
        <FixedAd
          adSlot="1111111111"
          width={300}
          height={250}
          className="mb-4"
        />

        {/* 其他側邊欄內容 */}
        <div className="widget">
          {/* ... */}
        </div>
      </div>
    </aside>
  )
}
```

## 21.6 處理 SSR/SSG 環境

### 21.6.1 僅在客戶端渲染廣告

由於 AdSense 腳本只能在客戶端運行，我們需要確保廣告只在客戶端渲染：

```tsx
// components/AdSense/ClientOnlyAd.tsx
import { useEffect, useState } from 'react'
import ResponsiveAd from './ResponsiveAd'

interface ClientOnlyAdProps {
  adSlot: string
  className?: string
}

export default function ClientOnlyAd({ adSlot, className }: ClientOnlyAdProps) {
  const [isMounted, setIsMounted] = useState(false)

  useEffect(() => {
    setIsMounted(true)
  }, [])

  // 僅在客戶端渲染廣告
  if (!isMounted) {
    return (
      <div className={`ad-placeholder ${className || ''}`} style={{ minHeight: '250px' }}>
        {/* 載入中的佔位符 */}
      </div>
    )
  }

  return <ResponsiveAd adSlot={adSlot} className={className} />
}
```

### 21.6.2 使用 Dynamic Import

```tsx
// pages/article/[id].tsx
import dynamic from 'next/dynamic'

// 動態載入廣告組件，禁用 SSR
const ClientOnlyAd = dynamic(
  () => import('@/components/AdSense/ClientOnlyAd'),
  {
    ssr: false,
    loading: () => (
      <div className="ad-placeholder" style={{ minHeight: '250px' }}>
        載入廣告中...
      </div>
    )
  }
)

export default function ArticlePage() {
  return (
    <article>
      <h1>文章標題</h1>
      <ClientOnlyAd adSlot="1234567890" className="my-8" />
      <p>文章內容...</p>
    </article>
  )
}
```

## 21.7 環境變數配置

### 21.7.1 設定環境變數

```bash
# .env.local
NEXT_PUBLIC_ADSENSE_ID=ca-pub-xxxxxxxxxxxxxxxx
NEXT_PUBLIC_ADSENSE_ENABLED=true
```

### 21.7.2 使用環境變數

```tsx
// lib/adsense.config.ts
export const adsenseConfig = {
  id: process.env.NEXT_PUBLIC_ADSENSE_ID || '',
  enabled: process.env.NEXT_PUBLIC_ADSENSE_ENABLED === 'true',
  slots: {
    headerBanner: '1234567890',
    articleTop: '2345678901',
    articleMiddle: '3456789012',
    articleBottom: '4567890123',
    sidebar: '5678901234',
    footer: '6789012345'
  }
}
```

```tsx
// pages/_app.tsx
import Script from 'next/script'
import { adsenseConfig } from '@/lib/adsense.config'

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <>
      {adsenseConfig.enabled && (
        <Script
          id="google-adsense"
          async
          src={`https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=${adsenseConfig.id}`}
          crossOrigin="anonymous"
          strategy="afterInteractive"
        />
      )}

      <Component {...pageProps} />
    </>
  )
}
```

## 21.8 自動廣告（Auto Ads）

### 21.8.1 啟用自動廣告

自動廣告會讓 Google 自動在您的頁面上放置廣告：

```tsx
// pages/_app.tsx
import Script from 'next/script'

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <>
      <Script
        id="google-adsense-auto"
        async
        src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-xxxxxxxxxxxxxxxx"
        crossOrigin="anonymous"
        strategy="afterInteractive"
        onLoad={() => {
          // 啟用自動廣告
          ;(window.adsbygoogle = window.adsbygoogle || []).push({
            google_ad_client: 'ca-pub-xxxxxxxxxxxxxxxx',
            enable_page_level_ads: true
          })
        }}
      />

      <Component {...pageProps} />
    </>
  )
}
```

### 21.8.2 手動與自動廣告混合使用

```tsx
// pages/_app.tsx
import Script from 'next/script'

function MyApp({ Component, pageProps }: AppProps) {
  return (
    <>
      {/* AdSense 腳本 */}
      <Script
        id="google-adsense"
        async
        src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-xxxxxxxxxxxxxxxx"
        crossOrigin="anonymous"
        strategy="afterInteractive"
      />

      {/* 可選：啟用自動廣告 */}
      <Script
        id="google-adsense-auto-config"
        strategy="afterInteractive"
        dangerouslySetInnerHTML={{
          __html: `
            (adsbygoogle = window.adsbygoogle || []).push({
              google_ad_client: "ca-pub-xxxxxxxxxxxxxxxx",
              enable_page_level_ads: true,
              overlays: {bottom: true}
            });
          `
        }}
      />

      <Component {...pageProps} />
    </>
  )
}
```

## 21.9 廣告效能追蹤

### 21.9.1 監控廣告載入

```tsx
// components/AdSense/TrackedAd.tsx
import { useEffect, useRef, useState } from 'react'
import AdUnit from './AdUnit'

interface TrackedAdProps {
  adSlot: string
  adName: string
}

export default function TrackedAd({ adSlot, adName }: TrackedAdProps) {
  const [loadTime, setLoadTime] = useState<number>(0)
  const startTimeRef = useRef<number>(0)

  useEffect(() => {
    startTimeRef.current = performance.now()

    const observer = new MutationObserver((mutations) => {
      mutations.forEach((mutation) => {
        if (mutation.type === 'childList' && mutation.addedNodes.length > 0) {
          const endTime = performance.now()
          const duration = endTime - startTimeRef.current
          setLoadTime(duration)

          // 發送追蹤資料到分析工具
          if (typeof window !== 'undefined' && window.gtag) {
            window.gtag('event', 'ad_loaded', {
              ad_name: adName,
              ad_slot: adSlot,
              load_time: duration
            })
          }
        }
      })
    })

    return () => observer.disconnect()
  }, [adSlot, adName])

  return (
    <div className="tracked-ad">
      <AdUnit adSlot={adSlot} />
      {process.env.NODE_ENV === 'development' && loadTime > 0 && (
        <div className="text-xs text-gray-500 mt-1">
          載入時間: {loadTime.toFixed(2)}ms
        </div>
      )}
    </div>
  )
}
```

### 21.9.2 錯誤處理

```tsx
// components/AdSense/SafeAd.tsx
import { useEffect, useState } from 'react'
import ResponsiveAd from './ResponsiveAd'

interface SafeAdProps {
  adSlot: string
  fallback?: React.ReactNode
}

export default function SafeAd({ adSlot, fallback }: SafeAdProps) {
  const [hasError, setHasError] = useState(false)

  useEffect(() => {
    const handleAdError = () => {
      setHasError(true)
      console.warn(`AdSense 廣告載入失敗: ${adSlot}`)
    }

    window.addEventListener('error', handleAdError)
    return () => window.removeEventListener('error', handleAdError)
  }, [adSlot])

  if (hasError && fallback) {
    return <>{fallback}</>
  }

  return <ResponsiveAd adSlot={adSlot} />
}
```

## 21.10 開發環境測試

### 21.10.1 測試模式配置

```tsx
// components/AdSense/DevAd.tsx
interface DevAdProps {
  adSlot: string
  className?: string
}

export default function DevAd({ adSlot, className }: DevAdProps) {
  const isDev = process.env.NODE_ENV === 'development'

  if (isDev) {
    // 開發環境顯示佔位符
    return (
      <div
        className={`dev-ad-placeholder ${className || ''}`}
        style={{
          background: '#f0f0f0',
          border: '2px dashed #ccc',
          padding: '20px',
          textAlign: 'center',
          minHeight: '250px',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center'
        }}
      >
        <div>
          <p className="font-bold">廣告位置</p>
          <p className="text-sm text-gray-600">Slot ID: {adSlot}</p>
        </div>
      </div>
    )
  }

  return <ResponsiveAd adSlot={adSlot} className={className} />
}
```

## 21.11 最佳實踐

### 21.11.1 廣告密度建議

- **部落格文章**：每 1000 字插入 1-2 個廣告
- **首頁**：最多 3-4 個廣告位
- **列表頁**：在列表項之間穿插廣告

### 21.11.2 效能優化

```tsx
// components/AdSense/LazyAd.tsx
import { useEffect, useRef, useState } from 'react'
import ResponsiveAd from './ResponsiveAd'

interface LazyAdProps {
  adSlot: string
  threshold?: number
}

export default function LazyAd({ adSlot, threshold = 0.1 }: LazyAdProps) {
  const [isVisible, setIsVisible] = useState(false)
  const adContainerRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true)
          observer.disconnect()
        }
      },
      { threshold }
    )

    if (adContainerRef.current) {
      observer.observe(adContainerRef.current)
    }

    return () => observer.disconnect()
  }, [threshold])

  return (
    <div ref={adContainerRef} style={{ minHeight: '250px' }}>
      {isVisible && <ResponsiveAd adSlot={adSlot} />}
    </div>
  )
}
```

### 21.11.3 合規性與隱私

```tsx
// components/AdSense/ConsentAwareAd.tsx
import { useEffect, useState } from 'react'
import ResponsiveAd from './ResponsiveAd'

export default function ConsentAwareAd({ adSlot }: { adSlot: string }) {
  const [hasConsent, setHasConsent] = useState(false)

  useEffect(() => {
    // 檢查使用者是否同意 Cookie
    const consent = localStorage.getItem('ad-consent')
    setHasConsent(consent === 'true')
  }, [])

  if (!hasConsent) {
    return (
      <div className="ad-consent-message">
        <p>我們需要您的同意以顯示個人化廣告</p>
        <button onClick={() => {
          localStorage.setItem('ad-consent', 'true')
          setHasConsent(true)
        }}>
          同意
        </button>
      </div>
    )
  }

  return <ResponsiveAd adSlot={adSlot} />
}
```

## 21.12 完整範例：部落格應用

```tsx
// pages/blog/[slug].tsx
import { GetStaticProps, GetStaticPaths } from 'next'
import dynamic from 'next/dynamic'
import { adsenseConfig } from '@/lib/adsense.config'

const ClientOnlyAd = dynamic(
  () => import('@/components/AdSense/ClientOnlyAd'),
  { ssr: false }
)

const LazyAd = dynamic(
  () => import('@/components/AdSense/LazyAd'),
  { ssr: false }
)

interface BlogPostProps {
  post: {
    slug: string
    title: string
    content: string
    excerpt: string
  }
}

export default function BlogPost({ post }: BlogPostProps) {
  return (
    <div className="container mx-auto px-4 py-8">
      <article className="max-w-4xl mx-auto">
        {/* 文章頂部廣告 */}
        <ClientOnlyAd
          adSlot={adsenseConfig.slots.articleTop}
          className="mb-8"
        />

        <h1 className="text-4xl font-bold mb-4">{post.title}</h1>

        <div className="prose lg:prose-xl">
          {/* 文章內容的前半部分 */}
          <div dangerouslySetInnerHTML={{
            __html: post.content.slice(0, post.content.length / 2)
          }} />

          {/* 文章中間廣告 - 使用 Lazy Loading */}
          <LazyAd
            adSlot={adsenseConfig.slots.articleMiddle}
            threshold={0.1}
          />

          {/* 文章內容的後半部分 */}
          <div dangerouslySetInnerHTML={{
            __html: post.content.slice(post.content.length / 2)
          }} />
        </div>

        {/* 文章底部廣告 */}
        <ClientOnlyAd
          adSlot={adsenseConfig.slots.articleBottom}
          className="mt-8"
        />
      </article>
    </div>
  )
}

export const getStaticPaths: GetStaticPaths = async () => {
  // 取得所有文章路徑
  return {
    paths: [],
    fallback: 'blocking'
  }
}

export const getStaticProps: GetStaticProps = async ({ params }) => {
  // 取得文章資料
  return {
    props: {
      post: {
        slug: params?.slug,
        title: '範例文章',
        content: '文章內容...',
        excerpt: '摘要...'
      }
    },
    revalidate: 3600
  }
}
```

## 21.13 常見問題

### Q1: 廣告不顯示怎麼辦？

**可能原因：**
1. AdSense 帳號尚未審核通過
2. 廣告腳本載入失敗
3. AdBlocker 阻擋廣告
4. 廣告單元 ID 錯誤

**解決方法：**
```tsx
// 加入除錯日誌
useEffect(() => {
  console.log('AdSense Script Loaded:', !!window.adsbygoogle)
  console.log('Ad Slot:', adSlot)
}, [])
```

### Q2: SSR 環境下廣告重複載入？

使用 `useEffect` 和狀態管理確保只在客戶端載入一次：

```tsx
const [adPushed, setAdPushed] = useState(false)

useEffect(() => {
  if (!adPushed && window.adsbygoogle) {
    ;(window.adsbygoogle = window.adsbygoogle || []).push({})
    setAdPushed(true)
  }
}, [adPushed])
```

### Q3: 如何測試 AdSense 整合？

1. 使用 Chrome DevTools 檢查網路請求
2. 檢查 Console 是否有錯誤訊息
3. 使用 AdSense 帳號中的「廣告審查中心」
4. 在開發環境使用佔位符模擬廣告

## 21.14 總結

本章介紹了如何在 Next.js 應用中整合 Google AdSense：

- ✅ 使用 Next.js Script 組件優化腳本載入
- ✅ 建立可重用的廣告組件
- ✅ 處理 SSR/SSG 環境下的廣告渲染
- ✅ 實作懶載入提升效能
- ✅ 遵循最佳實踐與合規要求

正確整合 AdSense 可以為您的網站帶來收入，同時保持良好的使用者體驗和頁面效能。
