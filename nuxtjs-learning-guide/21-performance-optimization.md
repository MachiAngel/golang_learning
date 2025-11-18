# 第 21 章：效能優化與最佳實踐

## 目錄
- [效能評估工具](#效能評估工具)
- [程式碼分割策略](#程式碼分割策略)
- [延遲載入 (Lazy Loading)](#延遲載入-lazy-loading)
- [圖片最佳化](#圖片最佳化)
- [字型最佳化](#字型最佳化)
- [CSS 最佳化](#css-最佳化)
- [JavaScript 最佳化](#javascript-最佳化)
- [快取策略](#快取策略)
- [CDN 使用](#cdn-使用)
- [預載入與預連接](#預載入與預連接)
- [Bundle Size 分析](#bundle-size-分析)
- [實際優化案例](#實際優化案例)
- [檢查清單](#檢查清單)

---

## 效能評估工具

### 1. Lighthouse

Lighthouse 是 Google 提供的網站效能分析工具，可測量：
- Performance（效能）
- Accessibility（無障礙性）
- Best Practices（最佳實踐）
- SEO（搜尋引擎優化）

**使用方式：**

```bash
# 方法 1：Chrome DevTools
# 開啟 Chrome DevTools > Lighthouse 標籤 > Generate report

# 方法 2：使用 CLI
npm install -g lighthouse

# 執行分析
lighthouse https://your-website.com --view

# 輸出 JSON 報告
lighthouse https://your-website.com --output json --output-path ./report.json

# 只測試效能
lighthouse https://your-website.com --only-categories=performance
```

**在 CI/CD 中整合：**

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI

on:
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      staticDistDir: '.output/public',
      numberOfRuns: 3
    },
    assert: {
      preset: 'lighthouse:recommended',
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'categories:accessibility': ['error', { minScore: 0.9 }],
        'categories:seo': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
      }
    },
    upload: {
      target: 'temporary-public-storage'
    }
  }
}
```

### 2. PageSpeed Insights

```bash
# 使用 PageSpeed Insights API
curl "https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=https://your-site.com&key=YOUR_API_KEY"
```

**整合到專案：**

```typescript
// utils/pagespeed.ts
export interface PageSpeedResult {
  score: number
  metrics: {
    fcp: number
    lcp: number
    cls: number
    tbt: number
    tti: number
  }
}

export async function analyzePageSpeed(url: string): Promise<PageSpeedResult> {
  const apiKey = process.env.PAGESPEED_API_KEY
  const apiUrl = `https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=${encodeURIComponent(url)}&key=${apiKey}`

  const response = await fetch(apiUrl)
  const data = await response.json()

  return {
    score: data.lighthouseResult.categories.performance.score * 100,
    metrics: {
      fcp: data.lighthouseResult.audits['first-contentful-paint'].numericValue,
      lcp: data.lighthouseResult.audits['largest-contentful-paint'].numericValue,
      cls: data.lighthouseResult.audits['cumulative-layout-shift'].numericValue,
      tbt: data.lighthouseResult.audits['total-blocking-time'].numericValue,
      tti: data.lighthouseResult.audits['interactive'].numericValue
    }
  }
}
```

### 3. WebPageTest

```bash
# 使用 WebPageTest API
npm install webpagetest

# 執行測試腳本
node scripts/webpagetest.js
```

```javascript
// scripts/webpagetest.js
const WebPageTest = require('webpagetest')
const wpt = new WebPageTest('www.webpagetest.org', process.env.WPT_API_KEY)

wpt.runTest('https://your-site.com', {
  location: 'Dulles:Chrome',
  connectivity: '4G',
  runs: 3,
  video: true
}, (err, result) => {
  if (err) {
    console.error(err)
    return
  }

  console.log('Test ID:', result.data.testId)
  console.log('Summary:', result.data.summary)
})
```

### 4. Chrome DevTools Performance

```typescript
// composables/usePerformanceMonitor.ts
export const usePerformanceMonitor = () => {
  const metrics = ref({
    fcp: 0,
    lcp: 0,
    fid: 0,
    cls: 0,
    ttfb: 0
  })

  const observePerformance = () => {
    if (process.server) return

    // First Contentful Paint
    const paintObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (entry.name === 'first-contentful-paint') {
          metrics.value.fcp = entry.startTime
        }
      }
    })
    paintObserver.observe({ entryTypes: ['paint'] })

    // Largest Contentful Paint
    const lcpObserver = new PerformanceObserver((list) => {
      const entries = list.getEntries()
      const lastEntry = entries[entries.length - 1]
      metrics.value.lcp = lastEntry.startTime
    })
    lcpObserver.observe({ entryTypes: ['largest-contentful-paint'] })

    // First Input Delay
    const fidObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        metrics.value.fid = entry.processingStart - entry.startTime
      }
    })
    fidObserver.observe({ entryTypes: ['first-input'] })

    // Cumulative Layout Shift
    let clsValue = 0
    const clsObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        if (!(entry as any).hadRecentInput) {
          clsValue += (entry as any).value
          metrics.value.cls = clsValue
        }
      }
    })
    clsObserver.observe({ entryTypes: ['layout-shift'] })

    // Time to First Byte
    const navigationEntry = performance.getEntriesByType('navigation')[0] as PerformanceNavigationTiming
    if (navigationEntry) {
      metrics.value.ttfb = navigationEntry.responseStart - navigationEntry.requestStart
    }
  }

  const sendToAnalytics = () => {
    // 發送到 Google Analytics 4
    if (typeof gtag !== 'undefined') {
      gtag('event', 'web_vitals', {
        fcp: Math.round(metrics.value.fcp),
        lcp: Math.round(metrics.value.lcp),
        fid: Math.round(metrics.value.fid),
        cls: metrics.value.cls.toFixed(3),
        ttfb: Math.round(metrics.value.ttfb)
      })
    }
  }

  onMounted(() => {
    observePerformance()

    // 頁面離開時發送數據
    window.addEventListener('beforeunload', sendToAnalytics)
  })

  return {
    metrics: readonly(metrics),
    sendToAnalytics
  }
}
```

---

## 程式碼分割策略

### 1. 路由層級分割

Nuxt 3 自動為每個頁面進行程式碼分割：

```
pages/
├── index.vue          → /_nuxt/pages/index.[hash].js
├── about.vue          → /_nuxt/pages/about.[hash].js
└── blog/
    ├── index.vue      → /_nuxt/pages/blog/index.[hash].js
    └── [slug].vue     → /_nuxt/pages/blog/[slug].[hash].js
```

### 2. 組件層級分割

使用 `defineAsyncComponent` 延遲載入組件：

```typescript
// 方法 1：基本用法
const HeavyComponent = defineAsyncComponent(() =>
  import('~/components/HeavyComponent.vue')
)

// 方法 2：帶載入狀態
const HeavyComponent = defineAsyncComponent({
  loader: () => import('~/components/HeavyComponent.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,
  timeout: 3000
})
```

**實際範例：**

```vue
<!-- pages/dashboard.vue -->
<template>
  <div class="dashboard">
    <h1>儀表板</h1>

    <!-- 立即載入的核心組件 -->
    <DashboardHeader />

    <!-- 延遲載入的重型組件 -->
    <ClientOnly>
      <Suspense>
        <template #default>
          <AnalyticsChart />
        </template>
        <template #fallback>
          <ChartSkeleton />
        </template>
      </Suspense>
    </ClientOnly>

    <!-- 條件性載入 -->
    <LazyDataTable v-if="showTable" />
  </div>
</template>

<script setup lang="ts">
// 使用 Lazy 前綴自動延遲載入
// Nuxt 會自動處理 components/DataTable.vue 的延遲載入
const showTable = ref(false)

onMounted(() => {
  // 主要內容載入後再載入次要組件
  setTimeout(() => {
    showTable.value = true
  }, 1000)
})
</script>
```

### 3. 第三方庫分割

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  vite: {
    build: {
      rollupOptions: {
        output: {
          manualChunks: {
            // 將大型庫獨立打包
            'lodash': ['lodash-es'],
            'chart': ['chart.js'],
            'editor': ['@tiptap/vue-3', '@tiptap/starter-kit'],
            'vendor': ['axios', 'dayjs']
          }
        }
      }
    }
  }
})
```

### 4. 動態導入

```typescript
// composables/useChartLibrary.ts
export const useChartLibrary = () => {
  const loadChart = async () => {
    const { Chart } = await import('chart.js')
    const {
      CategoryScale,
      LinearScale,
      PointElement,
      LineElement
    } = await import('chart.js')

    Chart.register(CategoryScale, LinearScale, PointElement, LineElement)

    return Chart
  }

  return {
    loadChart
  }
}
```

**使用範例：**

```vue
<template>
  <div>
    <canvas ref="chartCanvas"></canvas>
  </div>
</template>

<script setup lang="ts">
const chartCanvas = ref<HTMLCanvasElement | null>(null)
const { loadChart } = useChartLibrary()

onMounted(async () => {
  const Chart = await loadChart()

  if (chartCanvas.value) {
    new Chart(chartCanvas.value, {
      type: 'line',
      data: {
        // 圖表資料
      }
    })
  }
})
</script>
```

---

## 延遲載入 (Lazy Loading)

### 1. 圖片延遲載入

```vue
<!-- components/LazyImage.vue -->
<template>
  <div ref="container" class="lazy-image-container">
    <img
      v-if="isVisible"
      :src="src"
      :alt="alt"
      :loading="loading"
      @load="onLoad"
      @error="onError"
      class="lazy-image"
      :class="{ 'loaded': loaded }"
    />
    <div v-else class="placeholder" :style="placeholderStyle">
      <slot name="placeholder">
        <div class="spinner"></div>
      </slot>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Props {
  src: string
  alt: string
  loading?: 'lazy' | 'eager'
  aspectRatio?: string
}

const props = withDefaults(defineProps<Props>(), {
  loading: 'lazy',
  aspectRatio: '16/9'
})

const container = ref<HTMLElement | null>(null)
const isVisible = ref(false)
const loaded = ref(false)

const placeholderStyle = computed(() => ({
  aspectRatio: props.aspectRatio
}))

const onLoad = () => {
  loaded.value = true
}

const onError = () => {
  console.error(`圖片載入失敗: ${props.src}`)
}

onMounted(() => {
  const observer = new IntersectionObserver(
    (entries) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          isVisible.value = true
          observer.disconnect()
        }
      })
    },
    {
      rootMargin: '50px',
      threshold: 0.01
    }
  )

  if (container.value) {
    observer.observe(container.value)
  }

  onUnmounted(() => {
    observer.disconnect()
  })
})
</script>

<style scoped>
.lazy-image-container {
  position: relative;
  overflow: hidden;
}

.placeholder {
  background: #f0f0f0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.lazy-image {
  width: 100%;
  height: auto;
  opacity: 0;
  transition: opacity 0.3s ease-in-out;
}

.lazy-image.loaded {
  opacity: 1;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #3498db;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
</style>
```

### 2. 組件延遲載入

```typescript
// composables/useLazyComponent.ts
export const useLazyComponent = (
  componentLoader: () => Promise<any>,
  options = { rootMargin: '100px', threshold: 0.01 }
) => {
  const elementRef = ref<HTMLElement | null>(null)
  const component = shallowRef<any>(null)
  const isVisible = ref(false)

  onMounted(() => {
    if (!elementRef.value) return

    const observer = new IntersectionObserver(
      async (entries) => {
        for (const entry of entries) {
          if (entry.isIntersecting) {
            isVisible.value = true

            try {
              const loadedComponent = await componentLoader()
              component.value = loadedComponent.default || loadedComponent
            } catch (error) {
              console.error('組件載入失敗:', error)
            }

            observer.disconnect()
          }
        }
      },
      options
    )

    observer.observe(elementRef.value)

    onUnmounted(() => {
      observer.disconnect()
    })
  })

  return {
    elementRef,
    component,
    isVisible
  }
}
```

**使用範例：**

```vue
<template>
  <div ref="elementRef">
    <component v-if="component" :is="component" v-bind="$attrs" />
    <div v-else class="loading">載入中...</div>
  </div>
</template>

<script setup lang="ts">
const { elementRef, component } = useLazyComponent(
  () => import('~/components/HeavyChart.vue')
)
</script>
```

### 3. 模組延遲載入

```typescript
// composables/useMarkdownParser.ts
export const useMarkdownParser = () => {
  let markdownIt: any = null

  const parse = async (content: string) => {
    if (!markdownIt) {
      // 只在需要時載入
      const MarkdownIt = await import('markdown-it')
      markdownIt = new MarkdownIt.default({
        html: true,
        linkify: true,
        typographer: true
      })
    }

    return markdownIt.render(content)
  }

  return {
    parse
  }
}
```

---

## 圖片最佳化

### 1. 使用 Nuxt Image

```bash
npm install @nuxt/image
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/image'],

  image: {
    // 自定義圖片域名
    domains: ['cdn.example.com', 'images.unsplash.com'],

    // 圖片服務提供商
    provider: 'ipx', // 或 'cloudinary', 'imgix' 等

    // 預設圖片格式
    format: ['webp', 'avif', 'jpeg'],

    // 快取設定
    cache: {
      maxAge: 60 * 60 * 24 * 365 // 1 年
    },

    // 預設圖片尺寸
    screens: {
      xs: 320,
      sm: 640,
      md: 768,
      lg: 1024,
      xl: 1280,
      xxl: 1536
    }
  }
})
```

**基本使用：**

```vue
<template>
  <div>
    <!-- 自動優化 -->
    <NuxtImg
      src="/images/hero.jpg"
      alt="Hero Image"
      width="1200"
      height="600"
      loading="lazy"
    />

    <!-- 響應式圖片 -->
    <NuxtPicture
      src="/images/product.jpg"
      :img-attrs="{ alt: 'Product Image' }"
      sizes="xs:100vw sm:50vw md:400px"
      :modifiers="{ quality: 80 }"
    />

    <!-- 帶格式回退 -->
    <NuxtPicture
      src="/images/banner.jpg"
      format="avif,webp,jpeg"
      :img-attrs="{ alt: 'Banner' }"
    />

    <!-- 模糊預覽 -->
    <NuxtImg
      src="/images/gallery/photo-1.jpg"
      placeholder="/images/gallery/photo-1-placeholder.jpg"
      :placeholder-class="'blur'"
    />
  </div>
</template>

<style>
.blur {
  filter: blur(20px);
  transform: scale(1.1);
}
</style>
```

### 2. 響應式圖片組件

```vue
<!-- components/ResponsiveImage.vue -->
<template>
  <picture>
    <source
      v-for="(source, index) in sources"
      :key="index"
      :srcset="source.srcset"
      :media="source.media"
      :type="source.type"
    />
    <img
      :src="fallbackSrc"
      :alt="alt"
      :loading="loading"
      :decoding="decoding"
      :width="width"
      :height="height"
      @load="$emit('load')"
      @error="$emit('error')"
    />
  </picture>
</template>

<script setup lang="ts">
interface ImageSource {
  srcset: string
  media?: string
  type?: string
}

interface Props {
  src: string
  alt: string
  sources?: ImageSource[]
  loading?: 'lazy' | 'eager'
  decoding?: 'async' | 'sync' | 'auto'
  width?: number
  height?: number
}

const props = withDefaults(defineProps<Props>(), {
  sources: () => [],
  loading: 'lazy',
  decoding: 'async'
})

defineEmits(['load', 'error'])

const fallbackSrc = computed(() => props.src)
</script>
```

**使用範例：**

```vue
<template>
  <ResponsiveImage
    src="/images/hero.jpg"
    alt="Hero"
    :sources="[
      {
        srcset: '/images/hero.avif 1x, /images/hero@2x.avif 2x',
        type: 'image/avif'
      },
      {
        srcset: '/images/hero.webp 1x, /images/hero@2x.webp 2x',
        type: 'image/webp'
      },
      {
        srcset: '/images/hero.jpg 1x, /images/hero@2x.jpg 2x',
        type: 'image/jpeg'
      }
    ]"
    :width="1200"
    :height="600"
  />
</template>
```

### 3. 圖片壓縮自動化

```javascript
// scripts/optimize-images.js
const sharp = require('sharp')
const fs = require('fs/promises')
const path = require('path')

async function optimizeImages(inputDir, outputDir) {
  const files = await fs.readdir(inputDir)

  for (const file of files) {
    const inputPath = path.join(inputDir, file)
    const stat = await fs.stat(inputPath)

    if (stat.isDirectory()) {
      await optimizeImages(inputPath, path.join(outputDir, file))
      continue
    }

    if (!/\.(jpg|jpeg|png)$/i.test(file)) continue

    const outputPath = path.join(outputDir, file)
    await fs.mkdir(outputDir, { recursive: true })

    // 生成多種格式
    await Promise.all([
      // WebP
      sharp(inputPath)
        .webp({ quality: 80 })
        .toFile(outputPath.replace(/\.(jpg|jpeg|png)$/i, '.webp')),

      // AVIF
      sharp(inputPath)
        .avif({ quality: 70 })
        .toFile(outputPath.replace(/\.(jpg|jpeg|png)$/i, '.avif')),

      // 優化原格式
      sharp(inputPath)
        .jpeg({ quality: 85, progressive: true })
        .png({ compressionLevel: 9 })
        .toFile(outputPath)
    ])

    console.log(`優化完成: ${file}`)
  }
}

optimizeImages('./public/images/original', './public/images/optimized')
  .then(() => console.log('所有圖片優化完成'))
  .catch(console.error)
```

```json
// package.json
{
  "scripts": {
    "optimize:images": "node scripts/optimize-images.js"
  },
  "devDependencies": {
    "sharp": "^0.32.0"
  }
}
```

---

## 字型最佳化

### 1. 自託管字型

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    head: {
      link: [
        // 預連接到字型 CDN
        {
          rel: 'preconnect',
          href: 'https://fonts.googleapis.com'
        },
        {
          rel: 'preconnect',
          href: 'https://fonts.gstatic.com',
          crossorigin: 'anonymous'
        },
        // 預載入關鍵字型
        {
          rel: 'preload',
          href: '/fonts/NotoSansTC-Regular.woff2',
          as: 'font',
          type: 'font/woff2',
          crossorigin: 'anonymous'
        }
      ]
    }
  }
})
```

### 2. 字型顯示策略

```css
/* assets/css/fonts.css */
@font-face {
  font-family: 'Noto Sans TC';
  src: url('/fonts/NotoSansTC-Regular.woff2') format('woff2'),
       url('/fonts/NotoSansTC-Regular.woff') format('woff');
  font-weight: 400;
  font-style: normal;
  font-display: swap; /* 使用系統字型直到自定義字型載入完成 */
}

@font-face {
  font-family: 'Noto Sans TC';
  src: url('/fonts/NotoSansTC-Bold.woff2') format('woff2');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}

/* 僅載入需要的字重 */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/Inter-Variable.woff2') format('woff2-variations');
  font-weight: 100 900;
  font-display: swap;
}
```

### 3. 使用 @nuxtjs/google-fonts

```bash
npm install @nuxtjs/google-fonts
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxtjs/google-fonts'],

  googleFonts: {
    families: {
      'Noto Sans TC': [400, 700],
      'Inter': [400, 500, 600, 700]
    },
    display: 'swap',
    prefetch: true,
    preconnect: true,
    preload: true,
    // 下載字型到本地
    download: true,
    base64: false,
    inject: true,
    // 僅包含需要的字符集
    subsets: ['latin', 'chinese-traditional']
  }
})
```

### 4. 字型子集化

```bash
# 安裝 fonttools
pip install fonttools brotli

# 創建字型子集（僅包含常用字）
pyftsubset NotoSansTC-Regular.ttf \
  --text-file=common-chars.txt \
  --output-file=NotoSansTC-Regular-Subset.ttf \
  --flavor=woff2

# 或使用 glyphhanger
npm install -g glyphhanger

# 分析網站使用的字符
glyphhanger https://your-site.com

# 生成子集
glyphhanger --subset=NotoSansTC-Regular.ttf --formats=woff2,woff
```

### 5. 字型載入監控

```typescript
// composables/useFontLoading.ts
export const useFontLoading = () => {
  const fontsLoaded = ref(false)
  const loadingError = ref(false)

  const checkFontLoading = async () => {
    if (!process.client || !('fonts' in document)) return

    try {
      await document.fonts.ready
      fontsLoaded.value = true
      console.log('字型載入完成')

      // 添加類名以觸發 CSS 過渡
      document.documentElement.classList.add('fonts-loaded')
    } catch (error) {
      console.error('字型載入失敗:', error)
      loadingError.value = true
    }
  }

  onMounted(() => {
    checkFontLoading()
  })

  return {
    fontsLoaded: readonly(fontsLoaded),
    loadingError: readonly(loadingError)
  }
}
```

```css
/* 字型載入前後的樣式 */
body {
  font-family: system-ui, -apple-system, sans-serif;
  transition: font-family 0.3s ease;
}

.fonts-loaded body {
  font-family: 'Noto Sans TC', sans-serif;
}
```

---

## CSS 最佳化

### 1. 移除未使用的 CSS

```bash
npm install -D @fullhuman/postcss-purgecss
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  postcss: {
    plugins: {
      '@fullhuman/postcss-purgecss': process.env.NODE_ENV === 'production' ? {
        content: [
          './components/**/*.vue',
          './layouts/**/*.vue',
          './pages/**/*.vue',
          './plugins/**/*.ts',
          './app.vue'
        ],
        safelist: [
          // 保留動態類名
          /-(leave|enter|appear)(|-(to|from|active))$/,
          /^(?!cursor-move).+-move$/,
          /^router-link(|-exact)-active$/,
          /data-v-.*/
        ]
      } : false,
      autoprefixer: {}
    }
  }
})
```

### 2. 關鍵 CSS 內聯

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  experimental: {
    inlineSSRStyles: true
  },

  hooks: {
    'build:manifest': (manifest) => {
      // 內聯小於 14KB 的 CSS
      for (const key in manifest) {
        const file = manifest[key]
        if (file.isEntry && file.css) {
          file.css = file.css.filter(cssFile => {
            const size = fs.statSync(path.join('.nuxt/dist/client', cssFile)).size
            return size > 14000
          })
        }
      }
    }
  }
})
```

### 3. CSS Modules 與 Scoped Styles

```vue
<!-- 使用 CSS Modules -->
<template>
  <div :class="$style.container">
    <h1 :class="$style.title">標題</h1>
  </div>
</template>

<style module>
.container {
  max-width: 1200px;
  margin: 0 auto;
}

.title {
  font-size: 2rem;
  font-weight: bold;
}
</style>
```

```vue
<!-- 使用 Scoped Styles -->
<template>
  <div class="card">
    <h2 class="card-title">{{ title }}</h2>
  </div>
</template>

<style scoped>
.card {
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.card-title {
  margin: 0 0 10px;
}
</style>
```

### 4. 使用 CSS 變數

```css
/* assets/css/variables.css */
:root {
  /* 顏色 */
  --color-primary: #3b82f6;
  --color-secondary: #8b5cf6;
  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-danger: #ef4444;

  /* 間距 */
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;

  /* 字體 */
  --font-sans: 'Inter', system-ui, sans-serif;
  --font-mono: 'Fira Code', monospace;

  /* 斷點 */
  --breakpoint-sm: 640px;
  --breakpoint-md: 768px;
  --breakpoint-lg: 1024px;
  --breakpoint-xl: 1280px;
}

/* 深色模式 */
@media (prefers-color-scheme: dark) {
  :root {
    --color-bg: #1a1a1a;
    --color-text: #ffffff;
  }
}
```

### 5. 避免 CSS 阻塞渲染

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    head: {
      link: [
        // 非關鍵 CSS 使用 preload + print 技巧
        {
          rel: 'preload',
          href: '/css/non-critical.css',
          as: 'style',
          onload: "this.onload=null;this.rel='stylesheet'"
        },
        // 或使用 media 屬性
        {
          rel: 'stylesheet',
          href: '/css/print.css',
          media: 'print'
        }
      ]
    }
  }
})
```

---

## JavaScript 最佳化

### 1. Tree Shaking

```typescript
// ✅ 好的做法：具名導入
import { debounce, throttle } from 'lodash-es'

// ❌ 避免：導入整個庫
import _ from 'lodash'
```

```typescript
// utils/helpers.ts
// 使用具名導出，便於 tree shaking
export const formatDate = (date: Date) => {
  // ...
}

export const formatCurrency = (amount: number) => {
  // ...
}

// ❌ 避免默認導出大對象
// export default { formatDate, formatCurrency }
```

### 2. 代碼壓縮

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  vite: {
    build: {
      minify: 'terser',
      terserOptions: {
        compress: {
          drop_console: true, // 移除 console.log
          drop_debugger: true, // 移除 debugger
          pure_funcs: ['console.log', 'console.info'] // 移除特定函數
        },
        format: {
          comments: false // 移除註釋
        }
      }
    }
  }
})
```

### 3. 避免大型依賴

```typescript
// ❌ 避免：導入整個 moment.js (68KB)
import moment from 'moment'

// ✅ 好的做法：使用輕量級替代品
import dayjs from 'dayjs' // 僅 2KB

// 或使用原生 API
const formatDate = (date: Date) => {
  return new Intl.DateTimeFormat('zh-TW').format(date)
}
```

### 4. 防抖與節流

```typescript
// composables/useDebounce.ts
export const useDebounce = <T extends (...args: any[]) => any>(
  fn: T,
  delay: number = 300
) => {
  let timeoutId: NodeJS.Timeout | null = null

  const debouncedFn = (...args: Parameters<T>) => {
    if (timeoutId) {
      clearTimeout(timeoutId)
    }

    timeoutId = setTimeout(() => {
      fn(...args)
    }, delay)
  }

  const cancel = () => {
    if (timeoutId) {
      clearTimeout(timeoutId)
      timeoutId = null
    }
  }

  return {
    debouncedFn: debouncedFn as T,
    cancel
  }
}
```

```typescript
// composables/useThrottle.ts
export const useThrottle = <T extends (...args: any[]) => any>(
  fn: T,
  limit: number = 300
) => {
  let inThrottle: boolean = false

  const throttledFn = (...args: Parameters<T>) => {
    if (!inThrottle) {
      fn(...args)
      inThrottle = true
      setTimeout(() => {
        inThrottle = false
      }, limit)
    }
  }

  return throttledFn as T
}
```

**使用範例：**

```vue
<template>
  <input
    v-model="searchQuery"
    @input="handleSearch"
    placeholder="搜尋..."
  />
</template>

<script setup lang="ts">
const searchQuery = ref('')

const performSearch = (query: string) => {
  console.log('搜尋:', query)
  // 執行搜尋邏輯
}

const { debouncedFn: handleSearch } = useDebounce(
  () => performSearch(searchQuery.value),
  500
)
</script>
```

### 5. 虛擬滾動

```bash
npm install vue-virtual-scroller
```

```vue
<!-- components/VirtualList.vue -->
<template>
  <RecycleScroller
    class="scroller"
    :items="items"
    :item-size="60"
    key-field="id"
    v-slot="{ item }"
  >
    <div class="item">
      {{ item.name }}
    </div>
  </RecycleScroller>
</template>

<script setup lang="ts">
import { RecycleScroller } from 'vue-virtual-scroller'
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css'

interface Props {
  items: Array<{ id: number; name: string }>
}

defineProps<Props>()
</script>

<style scoped>
.scroller {
  height: 600px;
}

.item {
  height: 60px;
  padding: 12px;
  border-bottom: 1px solid #eee;
}
</style>
```

---

## 快取策略

### 1. HTTP 快取標頭

```typescript
// server/middleware/cache.ts
export default defineEventHandler((event) => {
  const url = event.node.req.url || ''

  // 靜態資源長期快取
  if (/\.(js|css|png|jpg|jpeg|gif|svg|woff|woff2)$/.test(url)) {
    setHeader(event, 'Cache-Control', 'public, max-age=31536000, immutable')
  }

  // HTML 短期快取
  else if (/\.(html)$/.test(url)) {
    setHeader(event, 'Cache-Control', 'public, max-age=3600, must-revalidate')
  }

  // API 不快取
  else if (url.startsWith('/api/')) {
    setHeader(event, 'Cache-Control', 'no-cache, no-store, must-revalidate')
  }
})
```

### 2. Service Worker 快取

```typescript
// public/sw.js
const CACHE_NAME = 'v1'
const urlsToCache = [
  '/',
  '/css/main.css',
  '/js/main.js',
  '/images/logo.png'
]

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(urlsToCache))
  )
})

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        // 快取命中則返回，否則發起網路請求
        return response || fetch(event.request)
      })
  )
})

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames.map((cacheName) => {
          if (cacheName !== CACHE_NAME) {
            return caches.delete(cacheName)
          }
        })
      )
    })
  )
})
```

```typescript
// plugins/sw.client.ts
export default defineNuxtPlugin(() => {
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker
        .register('/sw.js')
        .then((registration) => {
          console.log('SW 註冊成功:', registration.scope)
        })
        .catch((error) => {
          console.log('SW 註冊失敗:', error)
        })
    })
  }
})
```

### 3. 記憶體快取（Memoization）

```typescript
// utils/memoize.ts
export function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map()

  return ((...args: any[]) => {
    const key = JSON.stringify(args)

    if (cache.has(key)) {
      return cache.get(key)
    }

    const result = fn(...args)
    cache.set(key, result)

    return result
  }) as T
}
```

```typescript
// 使用範例
const expensiveCalculation = memoize((n: number) => {
  console.log('計算中...')
  return n * n
})

console.log(expensiveCalculation(5)) // 輸出: 計算中... 25
console.log(expensiveCalculation(5)) // 輸出: 25 (從快取中取得)
```

### 4. API 回應快取

```typescript
// composables/useCachedFetch.ts
export const useCachedFetch = <T>(
  url: string,
  options: {
    ttl?: number // 快取存活時間（毫秒）
    key?: string
  } = {}
) => {
  const { ttl = 60000, key = url } = options

  return useFetch<T>(url, {
    key,
    getCachedData(key) {
      const cached = nuxtApp.payload.data[key] || nuxtApp.static.data[key]

      if (!cached) return null

      const timestamp = nuxtApp.payload._timestamp || Date.now()
      const age = Date.now() - timestamp

      if (age > ttl) return null

      return cached
    }
  })
}
```

---

## CDN 使用

### 1. Cloudflare CDN 設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    cdnURL: process.env.CDN_URL || ''
  },

  nitro: {
    prerender: {
      crawlLinks: true,
      routes: ['/sitemap.xml']
    }
  }
})
```

```bash
# .env
CDN_URL=https://cdn.yourdomain.com
```

### 2. 靜態資源 CDN

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  vite: {
    build: {
      rollupOptions: {
        output: {
          assetFileNames: (assetInfo) => {
            let extType = assetInfo.name.split('.').pop()
            if (/png|jpe?g|svg|gif|tiff|bmp|ico/i.test(extType)) {
              extType = 'images'
            } else if (/woff|woff2|eot|ttf|otf/i.test(extType)) {
              extType = 'fonts'
            }
            return `${extType}/[name]-[hash][extname]`
          },
          chunkFileNames: 'js/[name]-[hash].js',
          entryFileNames: 'js/[name]-[hash].js'
        }
      }
    }
  }
})
```

### 3. 圖片 CDN

```vue
<template>
  <NuxtImg
    provider="cloudinary"
    src="/sample.jpg"
    width="800"
    height="600"
    :modifiers="{ quality: 80, format: 'auto' }"
  />
</template>
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  image: {
    cloudinary: {
      baseURL: 'https://res.cloudinary.com/your-cloud-name/image/upload/'
    },
    providers: {
      custom: {
        name: 'custom',
        provider: '~/providers/custom-image-provider.ts',
        options: {
          baseURL: 'https://your-cdn.com'
        }
      }
    }
  }
})
```

---

## 預載入與預連接

### 1. Resource Hints

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    head: {
      link: [
        // DNS 預解析
        {
          rel: 'dns-prefetch',
          href: 'https://api.example.com'
        },
        // 預連接
        {
          rel: 'preconnect',
          href: 'https://fonts.googleapis.com',
          crossorigin: 'anonymous'
        },
        // 預載入關鍵資源
        {
          rel: 'preload',
          href: '/fonts/main-font.woff2',
          as: 'font',
          type: 'font/woff2',
          crossorigin: 'anonymous'
        },
        // 預取得下一頁資源
        {
          rel: 'prefetch',
          href: '/about'
        }
      ]
    }
  }
})
```

### 2. 動態預取得

```vue
<!-- components/ArticleCard.vue -->
<template>
  <NuxtLink
    :to="`/articles/${article.slug}`"
    @mouseenter="prefetchArticle"
  >
    <h3>{{ article.title }}</h3>
  </NuxtLink>
</template>

<script setup lang="ts">
interface Props {
  article: {
    id: number
    slug: string
    title: string
  }
}

const props = defineProps<Props>()

const prefetchArticle = () => {
  // 預取得文章資料
  prefetchComponents(`/articles/${props.article.slug}`)

  // 預取得 API 資料
  $fetch(`/api/articles/${props.article.id}`)
}
</script>
```

### 3. 路由預載入

```typescript
// plugins/prefetch.client.ts
export default defineNuxtPlugin((nuxtApp) => {
  const router = useRouter()

  // 預取得所有路由組件
  router.getRoutes().forEach((route) => {
    if (route.components?.default) {
      // 延遲預取得，避免阻塞初始載入
      setTimeout(() => {
        route.components.default()
      }, 3000)
    }
  })
})
```

---

## Bundle Size 分析

### 1. Vite Bundle Analyzer

```bash
npm install -D rollup-plugin-visualizer
```

```typescript
// nuxt.config.ts
import { visualizer } from 'rollup-plugin-visualizer'

export default defineNuxtConfig({
  vite: {
    plugins: [
      visualizer({
        open: true,
        filename: 'dist/stats.html',
        gzipSize: true,
        brotliSize: true
      })
    ]
  }
})
```

### 2. Webpack Bundle Analyzer（Nuxt 2）

```bash
npm install -D webpack-bundle-analyzer
```

```javascript
// nuxt.config.js
export default {
  build: {
    analyze: process.env.ANALYZE === 'true'
  }
}
```

```bash
# 執行分析
ANALYZE=true npm run build
```

### 3. 自定義 Bundle 分析

```typescript
// scripts/analyze-bundle.ts
import { readFileSync } from 'fs'
import { gzipSync } from 'zlib'

const analyzeBundle = (filePath: string) => {
  const content = readFileSync(filePath)
  const gzipped = gzipSync(content)

  console.log(`文件: ${filePath}`)
  console.log(`原始大小: ${(content.length / 1024).toFixed(2)} KB`)
  console.log(`Gzip 大小: ${(gzipped.length / 1024).toFixed(2)} KB`)
  console.log(`壓縮率: ${((1 - gzipped.length / content.length) * 100).toFixed(2)}%`)
}

analyzeBundle('.output/public/_nuxt/entry.js')
```

---

## 實際優化案例

### 案例 1：部落格網站優化

**優化前：**
- Lighthouse Score: 65
- FCP: 3.2s
- LCP: 4.8s
- Bundle Size: 850KB

**優化策略：**

```typescript
// 1. 實現圖片延遲載入
<template>
  <article>
    <h1>{{ post.title }}</h1>
    <LazyImage :src="post.coverImage" :alt="post.title" />
    <div v-html="post.content"></div>
  </article>
</template>

// 2. 程式碼分割
const CommentSection = defineAsyncComponent(() =>
  import('~/components/CommentSection.vue')
)

// 3. 字型最佳化
// nuxt.config.ts
googleFonts: {
  families: {
    'Noto Sans TC': [400, 700]
  },
  download: true,
  display: 'swap'
}

// 4. 移除未使用的 CSS
postcss: {
  plugins: {
    '@fullhuman/postcss-purgecss': true
  }
}
```

**優化後：**
- Lighthouse Score: 95
- FCP: 1.2s
- LCP: 1.8s
- Bundle Size: 320KB

**效能提升：46%**

---

### 案例 2：電商網站優化

**優化前：**
- Lighthouse Score: 58
- TTI: 6.5s
- Total Bundle: 1.2MB

**優化策略：**

```typescript
// 1. 產品圖片優化
<NuxtPicture
  :src="product.image"
  format="avif,webp,jpg"
  :img-attrs="{ alt: product.name }"
  sizes="sm:100vw md:50vw lg:400px"
/>

// 2. 虛擬滾動商品列表
<RecycleScroller
  :items="products"
  :item-size="300"
  key-field="id"
  v-slot="{ item }"
>
  <ProductCard :product="item" />
</RecycleScroller>

// 3. API 回應快取
const { data: products } = await useCachedFetch('/api/products', {
  ttl: 300000 // 5 分鐘
})

// 4. 程式碼分割
const routes = [
  {
    path: '/checkout',
    component: () => import('~/pages/checkout.vue')
  }
]
```

**優化後：**
- Lighthouse Score: 92
- TTI: 2.8s
- Total Bundle: 450KB

**效能提升：57%**

---

## 檢查清單

### 基礎優化 ✅

- [ ] 啟用 Gzip/Brotli 壓縮
- [ ] 設定正確的 Cache-Control 標頭
- [ ] 使用 CDN 分發靜態資源
- [ ] 最小化 HTTP 請求數量
- [ ] 移除未使用的程式碼和依賴

### 圖片優化 ✅

- [ ] 使用現代圖片格式（WebP、AVIF）
- [ ] 實現響應式圖片
- [ ] 添加圖片延遲載入
- [ ] 設定適當的圖片尺寸
- [ ] 使用圖片壓縮工具

### CSS 優化 ✅

- [ ] 移除未使用的 CSS
- [ ] 內聯關鍵 CSS
- [ ] 使用 CSS Modules 或 Scoped Styles
- [ ] 避免 CSS 阻塞渲染
- [ ] 最小化 CSS 文件

### JavaScript 優化 ✅

- [ ] 實現程式碼分割
- [ ] 使用 Tree Shaking
- [ ] 延遲載入非關鍵組件
- [ ] 最小化和壓縮 JavaScript
- [ ] 避免大型第三方庫

### 字型優化 ✅

- [ ] 使用 font-display: swap
- [ ] 預載入關鍵字型
- [ ] 使用字型子集
- [ ] 自託管字型（避免外部請求）
- [ ] 使用可變字型

### 載入優化 ✅

- [ ] 使用 Resource Hints（preconnect、prefetch、preload）
- [ ] 實現 Service Worker 快取
- [ ] 使用 HTTP/2 或 HTTP/3
- [ ] 減少 DNS 查詢時間
- [ ] 優化伺服器回應時間（TTFB）

### 渲染優化 ✅

- [ ] 使用 SSR 或 SSG
- [ ] 實現骨架屏載入
- [ ] 避免佈局偏移（CLS）
- [ ] 優化關鍵渲染路徑
- [ ] 使用虛擬滾動處理長列表

### 監控與分析 ✅

- [ ] 設定效能監控（Lighthouse CI）
- [ ] 追蹤 Core Web Vitals
- [ ] 定期進行 Bundle Size 分析
- [ ] 使用 RUM（Real User Monitoring）
- [ ] 建立效能預算

---

## 效能預算範例

```javascript
// lighthouserc.js
module.exports = {
  ci: {
    assert: {
      preset: 'lighthouse:recommended',
      assertions: {
        // 效能分數至少 90
        'categories:performance': ['error', { minScore: 0.9 }],

        // 核心指標
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],

        // 資源大小
        'resource-summary:script:size': ['error', { maxNumericValue: 300000 }], // 300KB
        'resource-summary:stylesheet:size': ['error', { maxNumericValue: 50000 }], // 50KB
        'resource-summary:image:size': ['error', { maxNumericValue: 500000 }], // 500KB

        // 網路請求
        'network-requests': ['error', { maxNumericValue: 50 }]
      }
    }
  }
}
```

---

## 總結

效能優化是一個持續的過程，需要：

1. **定期監控**：使用 Lighthouse、PageSpeed Insights 等工具
2. **設定目標**：建立明確的效能預算
3. **逐步優化**：從影響最大的項目開始
4. **測量結果**：追蹤優化前後的指標變化
5. **持續改進**：根據用戶反饋和數據不斷調整

記住：**快速的網站 = 更好的用戶體驗 = 更高的轉換率**

恭喜！您已經掌握了 Nuxt.js 應用的效能優化技巧。
