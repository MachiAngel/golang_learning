# 第 20 章：Google AdSense 廣告整合

## 目錄
- [Google AdSense 申請流程](#google-adsense-申請流程)
- [驗證網站所有權](#驗證網站所有權)
- [AdSense 腳本整合](#adsense-腳本整合)
- [建立廣告組件](#建立廣告組件)
- [自動廣告設定](#自動廣告設定)
- [手動廣告單元](#手動廣告單元)
- [SSR 環境下的廣告載入](#ssr-環境下的廣告載入)
- [ClientOnly 包裝](#clientonly-包裝)
- [廣告位置最佳化](#廣告位置最佳化)
- [效能影響與優化](#效能影響與優化)
- [完整整合範例](#完整整合範例)
- [常見問題處理](#常見問題處理)

---

## Google AdSense 申請流程

### 1. 申請前準備

在申請 Google AdSense 之前，確保您的網站符合以下條件：

**基本要求：**
- 網站已上線並可公開訪問
- 內容原創且有價值
- 符合 Google AdSense 政策
- 有足夠的內容（建議至少 20-30 篇文章）
- 網站已運營至少 6 個月（某些地區）

**技術要求：**
- 擁有獨立域名（不能是免費子域名）
- 網站結構清晰，導航良好
- 頁面載入速度快
- 移動設備友好
- 有隱私權政策頁面

### 2. 申請步驟

```bash
# 步驟 1：訪問 AdSense 官網
# https://www.google.com/adsense/

# 步驟 2：點擊「開始使用」並登入 Google 帳號

# 步驟 3：填寫申請表單
# - 網站 URL
# - 電子郵件地址
# - 選擇是否接收個人化建議

# 步驟 4：同意條款並提交申請

# 步驟 5：將 AdSense 代碼添加到網站
```

---

## 驗證網站所有權

### 方法 1：使用 AdSense 代碼驗證

在申請 AdSense 後，您會收到一段驗證代碼，需要將其添加到網站的 `<head>` 標籤中。

**在 Nuxt.js 中實現：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    head: {
      script: [
        {
          src: 'https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js',
          'data-ad-client': 'ca-pub-XXXXXXXXXXXXXXXX', // 替換為您的發布商 ID
          async: true,
          crossorigin: 'anonymous'
        }
      ]
    }
  }
})
```

### 方法 2：使用 HTML 標籤驗證

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    head: {
      meta: [
        {
          name: 'google-adsense-account',
          content: 'ca-pub-XXXXXXXXXXXXXXXX'
        }
      ]
    }
  }
})
```

### 方法 3：使用 Google Tag Manager

如果您已經使用 GTM，可以透過 GTM 添加 AdSense 代碼。

---

## AdSense 腳本整合

### 使用 useHead 動態載入

```typescript
// composables/useAdSense.ts
export const useAdSense = () => {
  const config = useRuntimeConfig()

  const loadAdSense = () => {
    useHead({
      script: [
        {
          src: 'https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js',
          'data-ad-client': config.public.adsenseId,
          async: true,
          crossorigin: 'anonymous'
        }
      ]
    })
  }

  return {
    loadAdSense
  }
}
```

### 環境變數設定

```bash
# .env
NUXT_PUBLIC_ADSENSE_ID=ca-pub-XXXXXXXXXXXXXXXX
NUXT_PUBLIC_ADSENSE_TEST_MODE=false
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      adsenseId: process.env.NUXT_PUBLIC_ADSENSE_ID || '',
      adsenseTestMode: process.env.NUXT_PUBLIC_ADSENSE_TEST_MODE === 'true'
    }
  }
})
```

---

## 建立廣告組件

### 基礎廣告組件

```vue
<!-- components/AdSense/DisplayAd.vue -->
<template>
  <ClientOnly>
    <div class="adsense-container">
      <ins
        class="adsbygoogle"
        :style="adStyle"
        :data-ad-client="adClient"
        :data-ad-slot="adSlot"
        :data-ad-format="adFormat"
        :data-full-width-responsive="fullWidthResponsive"
      ></ins>
    </div>
  </ClientOnly>
</template>

<script setup lang="ts">
interface Props {
  adSlot: string
  adFormat?: string
  style?: string
  fullWidthResponsive?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  adFormat: 'auto',
  style: 'display:block',
  fullWidthResponsive: true
})

const config = useRuntimeConfig()
const adClient = config.public.adsenseId

const adStyle = computed(() => props.style)

onMounted(() => {
  try {
    // 推送廣告到 adsbygoogle 陣列
    (window.adsbygoogle = window.adsbygoogle || []).push({})
  } catch (error) {
    console.error('AdSense 載入錯誤:', error)
  }
})
</script>

<style scoped>
.adsense-container {
  margin: 20px 0;
  text-align: center;
}

.adsbygoogle {
  display: block;
}
</style>
```

### TypeScript 類型定義

```typescript
// types/adsense.d.ts
interface Window {
  adsbygoogle: any[]
}

export interface AdSenseConfig {
  client: string
  slot: string
  format?: 'auto' | 'fluid' | 'rectangle' | 'vertical' | 'horizontal'
  responsive?: boolean
  test?: boolean
}

export interface AdSenseSlots {
  header: string
  sidebar: string
  inArticle: string
  footer: string
}
```

---

## 自動廣告設定

### 啟用自動廣告

自動廣告會自動在您的網站上放置廣告，無需手動設定廣告單元。

```typescript
// plugins/adsense-auto.client.ts
export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()

  if (!config.public.adsenseId) {
    console.warn('AdSense ID 未設定')
    return
  }

  // 載入 AdSense 腳本
  const script = document.createElement('script')
  script.src = 'https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js'
  script.async = true
  script.crossOrigin = 'anonymous'
  script.setAttribute('data-ad-client', config.public.adsenseId)

  // 如果是測試模式，添加測試參數
  if (config.public.adsenseTestMode) {
    script.setAttribute('data-adtest', 'on')
  }

  document.head.appendChild(script)
})
```

### 自動廣告配置

在 AdSense 控制台中：
1. 前往「廣告」→「依網站顯示」
2. 選擇您的網站
3. 啟用「自動廣告」
4. 選擇廣告格式（重疊式廣告、錨定廣告、文字廣告等）
5. 預覽並應用設定

---

## 手動廣告單元

### 展示型廣告（Display Ads）

```vue
<!-- components/AdSense/DisplayAd.vue -->
<template>
  <ClientOnly>
    <div class="ad-wrapper">
      <div class="ad-label" v-if="showLabel">廣告</div>
      <ins
        class="adsbygoogle"
        style="display:block"
        :data-ad-client="config.public.adsenseId"
        :data-ad-slot="adSlot"
        data-ad-format="auto"
        data-full-width-responsive="true"
      ></ins>
    </div>
  </ClientOnly>
</template>

<script setup lang="ts">
interface Props {
  adSlot: string
  showLabel?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  showLabel: true
})

const config = useRuntimeConfig()

onMounted(() => {
  try {
    (window.adsbygoogle = window.adsbygoogle || []).push({})
  } catch (e) {
    console.error('AdSense error:', e)
  }
})
</script>

<style scoped>
.ad-wrapper {
  margin: 20px 0;
}

.ad-label {
  font-size: 12px;
  color: #999;
  text-align: center;
  margin-bottom: 5px;
}
</style>
```

### 文章內廣告（In-Article Ads）

```vue
<!-- components/AdSense/InArticleAd.vue -->
<template>
  <ClientOnly>
    <div class="in-article-ad">
      <ins
        class="adsbygoogle"
        style="display:block; text-align:center;"
        :data-ad-client="config.public.adsenseId"
        :data-ad-slot="adSlot"
        data-ad-layout="in-article"
        data-ad-format="fluid"
      ></ins>
    </div>
  </ClientOnly>
</template>

<script setup lang="ts">
interface Props {
  adSlot: string
}

const props = defineProps<Props>()
const config = useRuntimeConfig()

onMounted(() => {
  try {
    (window.adsbygoogle = window.adsbygoogle || []).push({})
  } catch (e) {
    console.error('AdSense error:', e)
  }
})
</script>

<style scoped>
.in-article-ad {
  margin: 30px 0;
  padding: 20px 0;
}
</style>
```

### 文章內摘要廣告（In-Feed Ads）

```vue
<!-- components/AdSense/InFeedAd.vue -->
<template>
  <ClientOnly>
    <div class="in-feed-ad">
      <ins
        class="adsbygoogle"
        style="display:block"
        :data-ad-client="config.public.adsenseId"
        :data-ad-slot="adSlot"
        data-ad-format="fluid"
        data-ad-layout-key="YOUR_LAYOUT_KEY"
      ></ins>
    </div>
  </ClientOnly>
</template>

<script setup lang="ts">
interface Props {
  adSlot: string
  layoutKey?: string
}

const props = defineProps<Props>()
const config = useRuntimeConfig()

onMounted(() => {
  try {
    (window.adsbygoogle = window.adsbygoogle || []).push({})
  } catch (e) {
    console.error('AdSense error:', e)
  }
})
</script>
```

### 多廣告單元組件

```vue
<!-- components/AdSense/AdUnit.vue -->
<template>
  <ClientOnly>
    <div :class="['ad-unit', `ad-unit--${format}`]">
      <div v-if="showLabel" class="ad-label">廣告</div>
      <ins
        class="adsbygoogle"
        :style="adStyle"
        :data-ad-client="config.public.adsenseId"
        :data-ad-slot="adSlot"
        :data-ad-format="adFormat"
        :data-full-width-responsive="fullWidthResponsive.toString()"
      ></ins>
    </div>
  </ClientOnly>
</template>

<script setup lang="ts">
interface Props {
  adSlot: string
  format?: 'display' | 'in-article' | 'in-feed' | 'horizontal' | 'vertical'
  showLabel?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  format: 'display',
  showLabel: true
})

const config = useRuntimeConfig()

const adFormat = computed(() => {
  switch (props.format) {
    case 'in-article':
      return 'fluid'
    case 'in-feed':
      return 'fluid'
    default:
      return 'auto'
  }
})

const adStyle = computed(() => {
  switch (props.format) {
    case 'horizontal':
      return 'display:inline-block;width:728px;height:90px'
    case 'vertical':
      return 'display:inline-block;width:160px;height:600px'
    default:
      return 'display:block'
  }
})

const fullWidthResponsive = computed(() => {
  return props.format === 'display' || props.format === 'in-article'
})

onMounted(() => {
  try {
    (window.adsbygoogle = window.adsbygoogle || []).push({})
  } catch (e) {
    console.error('AdSense error:', e)
  }
})
</script>

<style scoped>
.ad-unit {
  margin: 20px 0;
}

.ad-label {
  font-size: 11px;
  color: #999;
  text-align: center;
  margin-bottom: 8px;
  text-transform: uppercase;
  letter-spacing: 1px;
}

.ad-unit--horizontal,
.ad-unit--vertical {
  text-align: center;
}
</style>
```

---

## SSR 環境下的廣告載入

### 理解 SSR 與廣告載入的衝突

在 Nuxt.js 的 SSR 模式下，廣告代碼需要在客戶端執行，因為：
- AdSense 需要訪問 `window` 和 `document` 物件
- 廣告需要測量視窗大小和使用者行為
- 伺服器端無法執行廣告腳本

### 解決方案：僅客戶端載入

```typescript
// plugins/adsense.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  const config = useRuntimeConfig()

  // 確保只在客戶端執行
  if (process.client) {
    // 檢查 AdSense 腳本是否已載入
    if (document.querySelector('script[src*="adsbygoogle.js"]')) {
      return
    }

    const script = document.createElement('script')
    script.src = `https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=${config.public.adsenseId}`
    script.async = true
    script.crossOrigin = 'anonymous'

    script.onerror = () => {
      console.error('AdSense 腳本載入失敗')
    }

    script.onload = () => {
      console.log('AdSense 腳本載入成功')
    }

    document.head.appendChild(script)
  }
})
```

### 路由變更時重新載入廣告

```typescript
// composables/useAdSenseRefresh.ts
export const useAdSenseRefresh = () => {
  const router = useRouter()

  const refreshAds = () => {
    if (process.client && window.adsbygoogle) {
      try {
        // 清除現有廣告
        const ads = document.querySelectorAll('.adsbygoogle')
        ads.forEach((ad) => {
          if (ad.innerHTML) {
            ad.innerHTML = ''
          }
        })

        // 重新推送廣告
        ads.forEach(() => {
          (window.adsbygoogle = window.adsbygoogle || []).push({})
        })
      } catch (error) {
        console.error('廣告刷新失敗:', error)
      }
    }
  }

  // 監聽路由變化
  router.afterEach(() => {
    // 延遲執行，確保 DOM 已更新
    setTimeout(refreshAds, 100)
  })

  return {
    refreshAds
  }
}
```

---

## ClientOnly 包裝

### 基本用法

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <div class="blog-post">
    <h1>{{ post.title }}</h1>

    <!-- 文章開頭廣告 -->
    <ClientOnly>
      <AdSenseDisplayAd ad-slot="1234567890" />
      <template #fallback>
        <div class="ad-placeholder">廣告載入中...</div>
      </template>
    </ClientOnly>

    <div class="content" v-html="post.content"></div>

    <!-- 文章結尾廣告 -->
    <ClientOnly>
      <AdSenseDisplayAd ad-slot="0987654321" />
    </ClientOnly>
  </div>
</template>

<style scoped>
.ad-placeholder {
  min-height: 250px;
  background: #f5f5f5;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #999;
}
</style>
```

### 進階 ClientOnly 組件

```vue
<!-- components/AdSense/ClientOnlyAd.vue -->
<template>
  <ClientOnly>
    <div v-if="shouldShowAd" class="ad-container">
      <component :is="adComponent" v-bind="adProps" />
    </div>
    <template #fallback>
      <div v-if="showPlaceholder" :class="['ad-placeholder', placeholderClass]">
        <slot name="loading">
          <div class="loading-text">{{ loadingText }}</div>
        </slot>
      </div>
    </template>
  </ClientOnly>
</template>

<script setup lang="ts">
interface Props {
  adComponent: any
  adProps?: Record<string, any>
  showPlaceholder?: boolean
  placeholderClass?: string
  loadingText?: string
  delay?: number
}

const props = withDefaults(defineProps<Props>(), {
  showPlaceholder: true,
  placeholderClass: '',
  loadingText: '廣告載入中...',
  delay: 0
})

const shouldShowAd = ref(false)

onMounted(() => {
  if (props.delay > 0) {
    setTimeout(() => {
      shouldShowAd.value = true
    }, props.delay)
  } else {
    shouldShowAd.value = true
  }
})
</script>

<style scoped>
.ad-placeholder {
  min-height: 250px;
  background: #f9f9f9;
  display: flex;
  align-items: center;
  justify-content: center;
  border: 1px dashed #ddd;
  border-radius: 4px;
}

.loading-text {
  color: #999;
  font-size: 14px;
}
</style>
```

---

## 廣告位置最佳化

### 最佳廣告位置

根據用戶體驗和收益平衡，以下是建議的廣告位置：

**高價值位置（優先級高）：**
1. 文章標題下方（首屏內）
2. 文章內容中間
3. 側邊欄頂部
4. 文章結尾

**中等價值位置：**
5. 導航欄下方
6. 側邊欄中間
7. 相關文章區域上方

**低價值位置（謹慎使用）：**
8. 頁尾
9. 評論區上方

### 響應式廣告佈局

```vue
<!-- components/AdSense/ResponsiveAdLayout.vue -->
<template>
  <div class="responsive-ad-layout">
    <!-- 桌面版：側邊欄廣告 -->
    <ClientOnly>
      <div v-if="isDesktop" class="sidebar-ad">
        <AdSenseDisplayAd ad-slot="DESKTOP_SIDEBAR_SLOT" />
      </div>

      <!-- 移動版：文章內廣告 -->
      <div v-else class="mobile-ad">
        <AdSenseInArticleAd ad-slot="MOBILE_IN_ARTICLE_SLOT" />
      </div>
    </ClientOnly>
  </div>
</template>

<script setup lang="ts">
const isDesktop = ref(false)

onMounted(() => {
  const checkDevice = () => {
    isDesktop.value = window.innerWidth >= 1024
  }

  checkDevice()
  window.addEventListener('resize', checkDevice)

  onUnmounted(() => {
    window.removeEventListener('resize', checkDevice)
  })
})
</script>
```

### 按內容長度顯示廣告

```vue
<!-- pages/article/[id].vue -->
<template>
  <article class="article">
    <h1>{{ article.title }}</h1>

    <!-- 頂部廣告（所有文章） -->
    <ClientOnly>
      <AdSenseDisplayAd ad-slot="TOP_AD_SLOT" />
    </ClientOnly>

    <div class="content" v-html="article.content"></div>

    <!-- 中間廣告（僅長文章） -->
    <ClientOnly v-if="isLongArticle">
      <AdSenseInArticleAd ad-slot="MIDDLE_AD_SLOT" />
    </ClientOnly>

    <!-- 底部廣告（所有文章） -->
    <ClientOnly>
      <AdSenseDisplayAd ad-slot="BOTTOM_AD_SLOT" />
    </ClientOnly>
  </article>
</template>

<script setup lang="ts">
const route = useRoute()
const { data: article } = await useFetch(`/api/articles/${route.params.id}`)

const isLongArticle = computed(() => {
  if (!article.value) return false
  // 假設超過 1000 字為長文章
  const wordCount = article.value.content.replace(/<[^>]*>/g, '').length
  return wordCount > 1000
})
</script>
```

### 廣告頻率控制

```typescript
// composables/useAdFrequency.ts
export const useAdFrequency = () => {
  const MAX_ADS_PER_PAGE = 3
  const adCount = ref(0)

  const canShowAd = () => {
    return adCount.value < MAX_ADS_PER_PAGE
  }

  const incrementAdCount = () => {
    adCount.value++
  }

  const resetAdCount = () => {
    adCount.value = 0
  }

  return {
    canShowAd,
    incrementAdCount,
    resetAdCount,
    adCount: readonly(adCount)
  }
}
```

```vue
<!-- 使用範例 -->
<template>
  <div>
    <ClientOnly v-if="canShowAd()">
      <AdSenseDisplayAd
        ad-slot="SLOT_ID"
        @mounted="incrementAdCount()"
      />
    </ClientOnly>
  </div>
</template>

<script setup lang="ts">
const { canShowAd, incrementAdCount } = useAdFrequency()
</script>
```

---

## 效能影響與優化

### 延遲載入廣告

```typescript
// composables/useLazyAd.ts
export const useLazyAd = (elementRef: Ref<HTMLElement | null>) => {
  const isVisible = ref(false)
  const observer = ref<IntersectionObserver | null>(null)

  onMounted(() => {
    if (!elementRef.value) return

    observer.value = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            isVisible.value = true
            observer.value?.disconnect()
          }
        })
      },
      {
        rootMargin: '200px', // 提前 200px 開始載入
        threshold: 0.01
      }
    )

    observer.value.observe(elementRef.value)
  })

  onUnmounted(() => {
    observer.value?.disconnect()
  })

  return {
    isVisible
  }
}
```

```vue
<!-- components/AdSense/LazyAd.vue -->
<template>
  <div ref="adContainer" class="lazy-ad">
    <ClientOnly>
      <AdSenseDisplayAd v-if="isVisible" :ad-slot="adSlot" />
      <div v-else class="ad-placeholder"></div>
    </ClientOnly>
  </div>
</template>

<script setup lang="ts">
interface Props {
  adSlot: string
}

const props = defineProps<Props>()
const adContainer = ref<HTMLElement | null>(null)
const { isVisible } = useLazyAd(adContainer)
</script>

<style scoped>
.ad-placeholder {
  min-height: 250px;
  background: #f5f5f5;
}
</style>
```

### 使用 requestIdleCallback

```typescript
// composables/useIdleAdLoad.ts
export const useIdleAdLoad = () => {
  const loadAd = (callback: () => void) => {
    if ('requestIdleCallback' in window) {
      requestIdleCallback(callback, { timeout: 2000 })
    } else {
      // Fallback for browsers that don't support requestIdleCallback
      setTimeout(callback, 1)
    }
  }

  return {
    loadAd
  }
}
```

```vue
<!-- 使用範例 -->
<script setup lang="ts">
const { loadAd } = useIdleAdLoad()
const showAd = ref(false)

onMounted(() => {
  loadAd(() => {
    showAd.value = true
  })
})
</script>
```

### 預連接到廣告伺服器

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    head: {
      link: [
        // 預連接到 AdSense 伺服器
        {
          rel: 'preconnect',
          href: 'https://pagead2.googlesyndication.com'
        },
        {
          rel: 'preconnect',
          href: 'https://googleads.g.doubleclick.net'
        },
        {
          rel: 'dns-prefetch',
          href: 'https://adservice.google.com'
        }
      ]
    }
  }
})
```

### 廣告腳本異步載入

```typescript
// plugins/adsense-optimized.client.ts
export default defineNuxtPlugin(() => {
  const config = useRuntimeConfig()

  // 使用 async/defer 載入
  const loadAdSenseScript = () => {
    return new Promise((resolve, reject) => {
      const script = document.createElement('script')
      script.src = `https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=${config.public.adsenseId}`
      script.async = true
      script.defer = true
      script.crossOrigin = 'anonymous'

      script.onload = resolve
      script.onerror = reject

      document.head.appendChild(script)
    })
  }

  // 在頁面載入完成後載入
  if (document.readyState === 'complete') {
    loadAdSenseScript()
  } else {
    window.addEventListener('load', () => {
      loadAdSenseScript()
    })
  }
})
```

---

## 完整整合範例

### 專案結構

```
nuxt-adsense-project/
├── components/
│   └── AdSense/
│       ├── DisplayAd.vue          # 展示廣告
│       ├── InArticleAd.vue        # 文章內廣告
│       ├── InFeedAd.vue           # 摘要廣告
│       ├── LazyAd.vue             # 延遲載入廣告
│       └── AdUnit.vue             # 通用廣告單元
├── composables/
│   ├── useAdSense.ts              # AdSense 核心邏輯
│   ├── useAdFrequency.ts          # 廣告頻率控制
│   ├── useLazyAd.ts               # 延遲載入
│   └── useAdSenseRefresh.ts       # 廣告刷新
├── plugins/
│   └── adsense.client.ts          # AdSense 插件
├── layouts/
│   └── default.vue                # 預設佈局（含廣告）
└── pages/
    └── blog/
        └── [slug].vue             # 部落格文章頁面
```

### 完整的部落格文章範例

```vue
<!-- pages/blog/[slug].vue -->
<template>
  <div class="blog-container">
    <aside class="sidebar">
      <!-- 側邊欄頂部廣告 -->
      <ClientOnly>
        <LazyAd ad-slot="SIDEBAR_TOP_SLOT" />
      </ClientOnly>

      <!-- 側邊欄內容 -->
      <div class="sidebar-content">
        <h3>熱門文章</h3>
        <!-- ... -->
      </div>

      <!-- 側邊欄中間廣告 -->
      <ClientOnly v-if="canShowAd()">
        <LazyAd
          ad-slot="SIDEBAR_MIDDLE_SLOT"
          @mounted="incrementAdCount()"
        />
      </ClientOnly>
    </aside>

    <main class="main-content">
      <article class="article">
        <!-- 文章標題 -->
        <h1 class="article-title">{{ article.title }}</h1>

        <!-- 文章元資訊 -->
        <div class="article-meta">
          <span>{{ formatDate(article.createdAt) }}</span>
          <span>作者：{{ article.author }}</span>
        </div>

        <!-- 標題下廣告 -->
        <ClientOnly v-if="canShowAd()">
          <AdSenseDisplayAd
            ad-slot="ARTICLE_TOP_SLOT"
            @mounted="incrementAdCount()"
          />
        </ClientOnly>

        <!-- 文章內容 -->
        <div class="article-content">
          <!-- 前半部分內容 -->
          <div v-html="firstHalf"></div>

          <!-- 中間廣告（僅長文章） -->
          <ClientOnly v-if="isLongArticle && canShowAd()">
            <AdSenseInArticleAd
              ad-slot="ARTICLE_MIDDLE_SLOT"
              @mounted="incrementAdCount()"
            />
          </ClientOnly>

          <!-- 後半部分內容 -->
          <div v-html="secondHalf"></div>
        </div>

        <!-- 文章結尾廣告 -->
        <ClientOnly v-if="canShowAd()">
          <AdSenseDisplayAd
            ad-slot="ARTICLE_BOTTOM_SLOT"
            @mounted="incrementAdCount()"
          />
        </ClientOnly>

        <!-- 相關文章 -->
        <div class="related-articles">
          <h3>相關文章</h3>
          <div class="articles-grid">
            <ArticleCard
              v-for="related in relatedArticles"
              :key="related.id"
              :article="related"
            />
          </div>
        </div>
      </article>
    </main>
  </div>
</template>

<script setup lang="ts">
const route = useRoute()
const { data: article } = await useFetch(`/api/blog/${route.params.slug}`)
const { data: relatedArticles } = await useFetch(`/api/blog/${route.params.slug}/related`)

const { canShowAd, incrementAdCount, resetAdCount } = useAdFrequency()

// 重置廣告計數
onMounted(() => {
  resetAdCount()
})

// 判斷是否為長文章
const isLongArticle = computed(() => {
  if (!article.value) return false
  const wordCount = article.value.content.replace(/<[^>]*>/g, '').length
  return wordCount > 1500
})

// 分割內容（用於在中間插入廣告）
const firstHalf = computed(() => {
  if (!article.value) return ''
  const content = article.value.content
  const midPoint = Math.floor(content.length / 2)
  return content.substring(0, midPoint)
})

const secondHalf = computed(() => {
  if (!article.value) return ''
  const content = article.value.content
  const midPoint = Math.floor(content.length / 2)
  return content.substring(midPoint)
})

const formatDate = (date: string) => {
  return new Date(date).toLocaleDateString('zh-TW')
}

// SEO 設定
useHead({
  title: article.value?.title || '',
  meta: [
    { name: 'description', content: article.value?.excerpt || '' },
    { property: 'og:title', content: article.value?.title || '' },
    { property: 'og:description', content: article.value?.excerpt || '' }
  ]
})
</script>

<style scoped>
.blog-container {
  display: grid;
  grid-template-columns: 1fr 300px;
  gap: 40px;
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.main-content {
  min-width: 0;
}

.sidebar {
  position: sticky;
  top: 20px;
  height: fit-content;
}

.article-title {
  font-size: 2.5rem;
  margin-bottom: 1rem;
  line-height: 1.2;
}

.article-meta {
  color: #666;
  margin-bottom: 2rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #eee;
}

.article-content {
  font-size: 1.125rem;
  line-height: 1.8;
  margin-bottom: 3rem;
}

.related-articles {
  margin-top: 4rem;
  padding-top: 2rem;
  border-top: 2px solid #eee;
}

.articles-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 20px;
  margin-top: 1.5rem;
}

@media (max-width: 1024px) {
  .blog-container {
    grid-template-columns: 1fr;
  }

  .sidebar {
    position: static;
    order: -1;
  }
}
</style>
```

### AdSense 配置文件

```typescript
// config/adsense.config.ts
export interface AdSlotConfig {
  id: string
  name: string
  format?: 'display' | 'in-article' | 'in-feed'
}

export const adSlots: Record<string, AdSlotConfig> = {
  // 文章頁面廣告
  ARTICLE_TOP: {
    id: '1234567890',
    name: '文章頂部廣告',
    format: 'display'
  },
  ARTICLE_MIDDLE: {
    id: '2345678901',
    name: '文章中間廣告',
    format: 'in-article'
  },
  ARTICLE_BOTTOM: {
    id: '3456789012',
    name: '文章底部廣告',
    format: 'display'
  },

  // 側邊欄廣告
  SIDEBAR_TOP: {
    id: '4567890123',
    name: '側邊欄頂部廣告',
    format: 'display'
  },
  SIDEBAR_MIDDLE: {
    id: '5678901234',
    name: '側邊欄中間廣告',
    format: 'display'
  },

  // 首頁廣告
  HOME_HERO: {
    id: '6789012345',
    name: '首頁橫幅廣告',
    format: 'display'
  },
  HOME_FEED: {
    id: '7890123456',
    name: '首頁摘要廣告',
    format: 'in-feed'
  }
}

export const adsenseConfig = {
  publisherId: process.env.NUXT_PUBLIC_ADSENSE_ID || '',
  testMode: process.env.NODE_ENV === 'development',
  autoAds: true,
  maxAdsPerPage: 3,
  enableLazyLoad: true,
  lazyLoadMargin: '200px'
}
```

### 全域 AdSense 管理

```typescript
// composables/useGlobalAdSense.ts
import { adSlots, adsenseConfig } from '~/config/adsense.config'

export const useGlobalAdSense = () => {
  const config = useRuntimeConfig()
  const isLoaded = useState('adsense-loaded', () => false)
  const isTestMode = computed(() => adsenseConfig.testMode)

  const init = () => {
    if (process.server) return
    if (isLoaded.value) return

    const script = document.createElement('script')
    script.src = `https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=${config.public.adsenseId}`
    script.async = true
    script.crossOrigin = 'anonymous'

    if (isTestMode.value) {
      script.setAttribute('data-adtest', 'on')
    }

    script.onload = () => {
      isLoaded.value = true
      console.log('[AdSense] 腳本載入成功')
    }

    script.onerror = () => {
      console.error('[AdSense] 腳本載入失敗')
    }

    document.head.appendChild(script)
  }

  const getSlot = (slotKey: string) => {
    return adSlots[slotKey]
  }

  return {
    init,
    isLoaded: readonly(isLoaded),
    isTestMode,
    getSlot,
    slots: adSlots
  }
}
```

---

## 常見問題處理

### 1. 廣告不顯示

**可能原因：**
- AdSense 帳號尚未審核通過
- 廣告代碼配置錯誤
- 被廣告攔截器阻擋
- 網站流量不足

**解決方法：**

```typescript
// 檢測廣告攔截器
export const useAdBlockDetector = () => {
  const isAdBlockEnabled = ref(false)

  onMounted(() => {
    // 方法 1：檢測 adsbygoogle 是否被阻擋
    setTimeout(() => {
      if (typeof window.adsbygoogle === 'undefined') {
        isAdBlockEnabled.value = true
      }
    }, 1000)

    // 方法 2：創建假的廣告元素檢測
    const testAd = document.createElement('div')
    testAd.innerHTML = '&nbsp;'
    testAd.className = 'adsbox'
    document.body.appendChild(testAd)

    setTimeout(() => {
      if (testAd.offsetHeight === 0) {
        isAdBlockEnabled.value = true
      }
      testAd.remove()
    }, 100)
  })

  return {
    isAdBlockEnabled: readonly(isAdBlockEnabled)
  }
}
```

```vue
<!-- 提示用戶關閉廣告攔截器 -->
<template>
  <div v-if="isAdBlockEnabled" class="adblock-notice">
    <p>偵測到您使用廣告攔截器</p>
    <p>請將本站加入白名單以支持我們的創作</p>
  </div>
</template>

<script setup lang="ts">
const { isAdBlockEnabled } = useAdBlockDetector()
</script>
```

### 2. 廣告顯示空白

**解決方法：**

```typescript
// 廣告載入監控
export const useAdLoadMonitor = () => {
  const adStatus = ref<'loading' | 'loaded' | 'failed'>('loading')

  const monitorAd = (adElement: HTMLElement) => {
    const observer = new MutationObserver((mutations) => {
      mutations.forEach((mutation) => {
        if (mutation.type === 'childList' && adElement.children.length > 0) {
          adStatus.value = 'loaded'
          observer.disconnect()
        }
      })
    })

    observer.observe(adElement, {
      childList: true,
      subtree: true
    })

    // 10 秒後仍未載入則視為失敗
    setTimeout(() => {
      if (adStatus.value === 'loading') {
        adStatus.value = 'failed'
        observer.disconnect()
      }
    }, 10000)
  }

  return {
    adStatus: readonly(adStatus),
    monitorAd
  }
}
```

### 3. 在 SPA 中切換頁面廣告不更新

**解決方法：**

```typescript
// plugins/adsense-router.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  const router = useRouter()

  router.afterEach((to, from) => {
    if (to.path !== from.path) {
      // 延遲執行，確保新頁面已渲染
      setTimeout(() => {
        const ads = document.querySelectorAll('.adsbygoogle')
        ads.forEach((ad) => {
          // 只處理空的廣告容器
          if (!ad.hasChildNodes() || ad.innerHTML.trim() === '') {
            try {
              (window.adsbygoogle = window.adsbygoogle || []).push({})
            } catch (e) {
              console.error('AdSense 推送失敗:', e)
            }
          }
        })
      }, 500)
    }
  })
})
```

### 4. 違反 AdSense 政策

**常見違規：**
- 點擊自己的廣告
- 要求用戶點擊廣告
- 在禁止內容上顯示廣告
- 廣告與內容過於接近

**預防措施：**

```vue
<!-- 確保廣告與內容有明確區隔 -->
<template>
  <div class="content-wrapper">
    <div class="article-content">
      <!-- 文章內容 -->
    </div>

    <div class="ad-separator"></div>

    <div class="ad-section">
      <div class="ad-label">廣告</div>
      <ClientOnly>
        <AdSenseDisplayAd ad-slot="SLOT_ID" />
      </ClientOnly>
    </div>
  </div>
</template>

<style scoped>
.ad-separator {
  height: 20px;
  background: transparent;
}

.ad-section {
  padding: 20px 0;
  border-top: 1px solid #eee;
  border-bottom: 1px solid #eee;
}

.ad-label {
  font-size: 11px;
  color: #999;
  text-align: center;
  margin-bottom: 10px;
  text-transform: uppercase;
}
</style>
```

### 5. 測試模式設定

```typescript
// 開發環境使用測試廣告
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      adsenseId: process.env.NUXT_PUBLIC_ADSENSE_ID,
      adsenseTestMode: process.env.NODE_ENV === 'development'
    }
  }
})
```

```vue
<!-- 測試廣告顯示 -->
<template>
  <ClientOnly>
    <div class="test-ad" v-if="config.public.adsenseTestMode">
      <div class="test-label">測試廣告</div>
      <ins
        class="adsbygoogle"
        style="display:block"
        :data-ad-client="config.public.adsenseId"
        :data-ad-slot="adSlot"
        data-ad-format="auto"
        data-adtest="on"
        data-full-width-responsive="true"
      ></ins>
    </div>
    <AdSenseDisplayAd v-else :ad-slot="adSlot" />
  </ClientOnly>
</template>
```

### 6. 收益追蹤與優化

```typescript
// composables/useAdAnalytics.ts
export const useAdAnalytics = () => {
  const trackAdImpression = (adSlot: string, position: string) => {
    // 發送到 GA4
    if (typeof gtag !== 'undefined') {
      gtag('event', 'ad_impression', {
        ad_slot: adSlot,
        ad_position: position,
        page_path: window.location.pathname
      })
    }
  }

  const trackAdClick = (adSlot: string) => {
    if (typeof gtag !== 'undefined') {
      gtag('event', 'ad_click', {
        ad_slot: adSlot,
        page_path: window.location.pathname
      })
    }
  }

  return {
    trackAdImpression,
    trackAdClick
  }
}
```

---

## 總結

### 最佳實踐檢查清單

✅ **設定階段：**
- [ ] AdSense 帳號已審核通過
- [ ] 發布商 ID 正確配置
- [ ] 環境變數設定完成
- [ ] 測試模式在開發環境啟用

✅ **技術實現：**
- [ ] 使用 ClientOnly 包裝廣告
- [ ] 實現延遲載入機制
- [ ] 配置預連接資源
- [ ] 處理路由切換時的廣告刷新

✅ **用戶體驗：**
- [ ] 廣告與內容明確區隔
- [ ] 不影響頁面載入速度
- [ ] 移動設備友好
- [ ] 廣告數量適中（每頁 3-5 個）

✅ **政策遵守：**
- [ ] 不誘導點擊廣告
- [ ] 廣告標示清楚
- [ ] 符合內容政策
- [ ] 定期檢查違規通知

✅ **效能優化：**
- [ ] 使用異步載入
- [ ] 實現廣告延遲載入
- [ ] 監控頁面效能指標
- [ ] 定期分析廣告表現

### 進階學習資源

- [Google AdSense 官方文檔](https://support.google.com/adsense)
- [AdSense 政策中心](https://support.google.com/adsense/answer/48182)
- [Web Vitals 優化指南](https://web.dev/vitals/)
- [Nuxt 3 官方文檔](https://nuxt.com/docs)

---

恭喜！您已經學會如何在 Nuxt.js 中整合 Google AdSense 廣告系統。記得要遵守 AdSense 政策，並持續優化廣告位置和效能表現。
