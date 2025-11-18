# 18. 部署到 Cloudflare Pages

## Cloudflare Pages 簡介

Cloudflare Pages 是 Cloudflare 提供的 JAMstack 平台，結合了強大的 CDN 和 Edge Computing 能力，特別適合部署 Nuxt.js 應用。

### 主要優勢

- ✅ **全球 CDN**：遍布全球 300+ 個節點
- ✅ **免費方案慷慨**：無限網站、無限流量
- ✅ **Edge Functions**：在邊緣運行服務器代碼
- ✅ **快速部署**：平均 30 秒內完成
- ✅ **自動 HTTPS**：免費 SSL 證書
- ✅ **Preview 部署**：每個分支都有獨立預覽
- ✅ **D1 資料庫**：Edge 上的 SQLite 資料庫
- ✅ **R2 儲存**：S3 相容的對象儲存
- ✅ **Workers KV**：全球分佈的鍵值存儲

### Cloudflare 生態系統

```
Cloudflare Pages  →  靜態網站託管 + SSR
Cloudflare Workers →  Edge Computing（Serverless Functions）
Cloudflare D1     →  Edge 資料庫（SQLite）
Cloudflare R2     →  對象儲存（類似 S3）
Cloudflare KV     →  鍵值存儲
Cloudflare Durable Objects → 狀態化 Edge Computing
```

### 適用場景

- 全球化網站（需要低延遲）
- 靜態網站和 SSG 應用
- Edge SSR 應用
- 高流量網站（免費無限流量）
- 需要 Edge Computing 的應用

---

## 支援的渲染模式

Cloudflare Pages 支援多種 Nuxt.js 渲染模式：

### 1. 靜態站點生成 (SSG)

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages',
    prerender: {
      crawlLinks: true,
      routes: [
        '/',
        '/about',
        '/blog',
        '/contact'
      ]
    }
  }
})
```

### 2. 服務器端渲染 (SSR) - Edge

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages'
  },

  routeRules: {
    // 動態 SSR 頁面
    '/blog/**': { ssr: true },

    // 靜態頁面
    '/': { prerender: true },
    '/about': { prerender: true },

    // SPA 模式
    '/dashboard/**': { ssr: false }
  }
})
```

### 3. 混合渲染（推薦）

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages',
    prerender: {
      crawlLinks: true,
      routes: ['/']
    }
  },

  routeRules: {
    // 首頁：預渲染
    '/': { prerender: true },

    // 部落格：ISR（增量靜態再生）
    '/blog/**': { swr: 3600 }, // 1 小時

    // API：Edge SSR
    '/api/**': { ssr: true },

    // 使用者儀表板：SPA
    '/dashboard/**': { ssr: false },

    // 商品頁：按需 ISR
    '/products/**': { isr: true }
  }
})
```

---

## 前置準備

### 1. 註冊 Cloudflare 帳號

訪問 [https://dash.cloudflare.com/sign-up](https://dash.cloudflare.com/sign-up) 註冊帳號。

### 2. 準備 Nuxt.js 項目

確保你的 `package.json` 正確配置：

```json
{
  "name": "my-nuxt-app",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "nuxt build",
    "dev": "nuxt dev",
    "generate": "nuxt generate",
    "preview": "nuxt preview"
  },
  "dependencies": {
    "nuxt": "^3.13.0",
    "vue": "^3.4.0"
  },
  "devDependencies": {
    "@cloudflare/workers-types": "^4.20241127.0"
  }
}
```

### 3. 安裝 Cloudflare Workers Types

```bash
# 安裝類型定義（支援 TypeScript）
npm install -D @cloudflare/workers-types

# 或使用 pnpm
pnpm add -D @cloudflare/workers-types
```

### 4. 配置 Nuxt for Cloudflare

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages'
  },

  // 運行時配置
  runtimeConfig: {
    // 私有環境變數（僅服務器端）
    apiSecret: '',

    public: {
      // 公開環境變數（客戶端也可訪問）
      apiBase: process.env.NUXT_PUBLIC_API_BASE || ''
    }
  },

  // 相容性設定
  compatibilityDate: '2024-11-01'
})
```

---

## Git 整合部署（推薦）

### 步驟 1：推送代碼到 Git

```bash
# 初始化 Git（如果還沒有）
git init

# 建立 .gitignore
cat > .gitignore << EOF
node_modules
.nuxt
.output
dist
.env
.DS_Store
.wrangler
EOF

# 提交代碼
git add .
git commit -m "Initial commit"

# 推送到 GitHub/GitLab
git remote add origin https://github.com/your-username/your-repo.git
git push -u origin main
```

### 步驟 2：在 Cloudflare Dashboard 建立項目

1. 訪問 [Cloudflare Pages](https://dash.cloudflare.com/pages)
2. 點擊 **"Create a project"**
3. 點擊 **"Connect to Git"**
4. 選擇 **GitHub** 或 **GitLab**
5. 授權 Cloudflare 訪問你的倉庫
6. 選擇你的 Nuxt.js 倉庫

### 步驟 3：配置構建設定

Cloudflare 會自動檢測 Nuxt.js 項目，但你可以自定義：

```yaml
Production branch: main

Build settings:
  Framework preset: Nuxt.js
  Build command: npm run build
  Build output directory: .output/public
  Root directory: /
  Environment variables: (可選)
    NODE_VERSION=18
    NUXT_PUBLIC_API_BASE=https://api.example.com
```

### 步驟 4：部署

點擊 **"Save and Deploy"**，Cloudflare 會：

1. Clone 你的倉庫
2. 安裝依賴（npm install）
3. 執行構建命令（npm run build）
4. 部署到全球 CDN
5. 提供預覽 URL

部署完成後：

```
✓ Success! Your site is live at:
  https://my-nuxt-app.pages.dev
```

---

## Wrangler CLI 部署

Wrangler 是 Cloudflare 的官方 CLI 工具。

### 安裝 Wrangler

```bash
# 全局安裝
npm install -g wrangler

# 或使用 pnpm
pnpm add -g wrangler

# 或作為項目依賴
npm install -D wrangler
```

### 登入 Cloudflare

```bash
# 登入（會開啟瀏覽器）
wrangler login

# 查看帳號資訊
wrangler whoami
```

### 初始化項目

```bash
# 建立 wrangler.toml（如果還沒有）
cat > wrangler.toml << EOF
name = "my-nuxt-app"
compatibility_date = "2024-11-01"
pages_build_output_dir = ".output/public"

[env.production]
routes = [
  { pattern = "example.com", custom_domain = true }
]

[env.production.vars]
ENVIRONMENT = "production"
EOF
```

### 完整的 wrangler.toml 配置

```toml
# wrangler.toml
name = "my-nuxt-app"
compatibility_date = "2024-11-01"
compatibility_flags = ["nodejs_compat"]

# 構建輸出目錄
pages_build_output_dir = ".output/public"

# 環境變數（非敏感資訊）
[vars]
ENVIRONMENT = "production"

# KV 命名空間（可選）
[[kv_namespaces]]
binding = "MY_KV"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# D1 資料庫（可選）
[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# R2 Bucket（可選）
[[r2_buckets]]
binding = "MY_BUCKET"
bucket_name = "my-bucket"

# Durable Objects（可選）
[[durable_objects.bindings]]
name = "MY_DO"
class_name = "MyDurableObject"
script_name = "my-durable-object"

# 路由配置
[[routes]]
pattern = "example.com/*"
custom_domain = true

[[routes]]
pattern = "www.example.com/*"
custom_domain = true
```

### 本地開發

```bash
# 使用 Nuxt 開發服務器
npm run dev

# 或使用 Wrangler（更接近生產環境）
wrangler pages dev .output/public --compatibility-date=2024-11-01

# 構建並本地預覽
npm run build
wrangler pages dev .output/public
```

### 部署到 Cloudflare Pages

```bash
# 構建項目
npm run build

# 部署到 Cloudflare Pages
wrangler pages deploy .output/public

# 部署到特定項目
wrangler pages deploy .output/public --project-name=my-nuxt-app

# 部署到特定分支
wrangler pages deploy .output/public --branch=staging
```

### 查看部署狀態

```bash
# 列出所有部署
wrangler pages deployments list

# 查看特定部署
wrangler pages deployment view DEPLOYMENT_ID

# 查看項目資訊
wrangler pages project list
```

---

## 環境變數設定

### 在 Cloudflare Dashboard 設定

1. 進入你的 Pages 項目
2. 點擊 **"Settings"** → **"Environment variables"**
3. 添加變數：

```
Production:
  NODE_VERSION = 18
  NUXT_PUBLIC_API_BASE = https://api.example.com
  API_SECRET = your-secret-key (Encrypted)

Preview:
  NUXT_PUBLIC_API_BASE = https://api-staging.example.com
  API_SECRET = staging-secret-key (Encrypted)
```

### 使用 Wrangler 設定

```bash
# 設定環境變數（非敏感）
wrangler pages project set-env \
  --project-name=my-nuxt-app \
  --env=production \
  NUXT_PUBLIC_API_BASE=https://api.example.com

# 設定 Secret（敏感資訊）
echo "your-secret-key" | wrangler pages secret put API_SECRET

# 列出所有 Secrets
wrangler pages secret list

# 刪除 Secret
wrangler pages secret delete API_SECRET
```

### 在代碼中使用環境變數

```typescript
// composables/useEnv.ts
export const useEnv = () => {
  const config = useRuntimeConfig()

  return {
    apiBase: config.public.apiBase,
    apiSecret: config.apiSecret // 僅服務器端可訪問
  }
}

// server/api/hello.ts
export default defineEventHandler((event) => {
  const config = useRuntimeConfig()
  const apiSecret = config.apiSecret

  return {
    message: 'Hello from Cloudflare Pages!',
    env: process.env.CF_PAGES ? 'Cloudflare Pages' : 'Local'
  }
})
```

### Cloudflare 系統環境變數

Cloudflare 自動提供以下環境變數：

```typescript
// 在服務器端可訪問
process.env.CF_PAGES // "1" 表示在 Cloudflare Pages 上運行
process.env.CF_PAGES_COMMIT_SHA // Git commit SHA
process.env.CF_PAGES_BRANCH // Git 分支名稱
process.env.CF_PAGES_URL // 部署 URL
```

---

## Functions 與 Workers

### Cloudflare Pages Functions

Pages Functions 自動將你的 Nuxt.js API routes 轉換為 Cloudflare Workers。

```typescript
// server/api/hello.ts
export default defineEventHandler((event) => {
  return {
    message: 'Hello from Edge!',
    timestamp: new Date().toISOString()
  }
})
```

訪問：`https://your-site.pages.dev/api/hello`

### 使用 Cloudflare Bindings

#### KV (Key-Value Store)

```typescript
// server/api/kv-example.ts
export default defineEventHandler(async (event) => {
  // 從 Cloudflare context 獲取 KV
  const kv = event.context.cloudflare.env.MY_KV

  // 寫入數據
  await kv.put('myKey', 'myValue', {
    expirationTtl: 3600 // 1 小時後過期
  })

  // 讀取數據
  const value = await kv.get('myKey')

  return { value }
})

// 配置 wrangler.toml
// [[kv_namespaces]]
// binding = "MY_KV"
// id = "your-kv-namespace-id"
```

#### D1 (SQLite Database)

```typescript
// server/api/db-example.ts
export default defineEventHandler(async (event) => {
  const db = event.context.cloudflare.env.DB

  // 執行查詢
  const result = await db.prepare(
    'SELECT * FROM users WHERE id = ?'
  ).bind(1).first()

  return result
})

// 建立資料庫
// wrangler d1 create my-database
// wrangler d1 execute my-database --command "CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)"
```

#### R2 (Object Storage)

```typescript
// server/api/r2-example.ts
export default defineEventHandler(async (event) => {
  const bucket = event.context.cloudflare.env.MY_BUCKET

  // 上傳文件
  await bucket.put('myfile.txt', 'Hello, R2!', {
    httpMetadata: {
      contentType: 'text/plain'
    }
  })

  // 讀取文件
  const object = await bucket.get('myfile.txt')
  const text = await object?.text()

  return { text }
})
```

### 完整的 Bindings 範例

```typescript
// server/api/bindings.ts
export default defineEventHandler(async (event) => {
  const env = event.context.cloudflare.env

  // KV
  const kvValue = await env.MY_KV.get('key')

  // D1
  const dbResult = await env.DB
    .prepare('SELECT * FROM users LIMIT 10')
    .all()

  // R2
  const r2Object = await env.MY_BUCKET.get('file.txt')

  return {
    kv: kvValue,
    db: dbResult.results,
    r2: await r2Object?.text()
  }
})
```

### TypeScript 支援

```typescript
// env.d.ts
/// <reference types="@cloudflare/workers-types" />

declare module 'h3' {
  interface H3EventContext {
    cloudflare: {
      request: Request
      env: {
        MY_KV: KVNamespace
        DB: D1Database
        MY_BUCKET: R2Bucket
      }
      context: ExecutionContext
    }
  }
}

export {}
```

---

## 自定義域名

### 添加自定義域名

#### 方法 1：通過 Dashboard

1. 進入 Pages 項目
2. 點擊 **"Custom domains"**
3. 點擊 **"Set up a custom domain"**
4. 輸入域名：`example.com` 或 `www.example.com`
5. 點擊 **"Continue"**

#### 方法 2：使用 Wrangler

```bash
# 添加自定義域名
wrangler pages domain add example.com --project-name=my-nuxt-app

# 列出域名
wrangler pages domain list --project-name=my-nuxt-app
```

### 配置 DNS

如果你的域名在 Cloudflare 上：

**選項 A：自動配置（推薦）**
- Cloudflare 會自動添加 DNS 記錄

**選項 B：手動配置**
```
Type: CNAME
Name: www（或 @）
Target: my-nuxt-app.pages.dev
Proxy status: Proxied（橙色雲朵）
```

如果域名不在 Cloudflare：

```
Type: CNAME
Name: www
Value: my-nuxt-app.pages.dev

Type: A (for apex domain)
Name: @
Value:
  - Cloudflare IP 地址（Cloudflare 會提供）
```

### SSL/TLS 設定

Cloudflare Pages 自動提供免費 SSL 證書：

1. 進入 **SSL/TLS** → **Overview**
2. 選擇加密模式：
   - **Flexible**：瀏覽器到 Cloudflare 加密
   - **Full**：端到端加密（推薦）
   - **Full (strict)**：嚴格證書驗證

### 重定向設定

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // www 重定向到非 www
    'https://www.example.com/**': {
      redirect: {
        to: 'https://example.com/**',
        statusCode: 301
      }
    }
  }
})
```

或使用 `_redirects` 文件：

```
# public/_redirects
https://www.example.com/* https://example.com/:splat 301
/old-page /new-page 301
/blog/* /articles/:splat 302
```

---

## Analytics 與效能

### Web Analytics

Cloudflare 提供免費的 Web Analytics，無需 Cookie，保護隱私。

#### 啟用 Analytics

1. 進入 Pages 項目
2. 點擊 **"Analytics"** → **"Web Analytics"**
3. 點擊 **"Enable Web Analytics"**

#### 手動添加（可選）

```vue
<!-- app.vue -->
<template>
  <div>
    <NuxtPage />
  </div>
</template>

<script setup>
useHead({
  script: [
    {
      src: 'https://static.cloudflareinsights.com/beacon.min.js',
      defer: true,
      'data-cf-beacon': JSON.stringify({
        token: 'your-analytics-token'
      })
    }
  ]
})
</script>
```

### 監控指標

Cloudflare Analytics 提供：

- **頁面瀏覽量**：總訪問量
- **獨立訪客**：唯一訪客數
- **Core Web Vitals**：
  - LCP (Largest Contentful Paint)
  - FID (First Input Delay)
  - CLS (Cumulative Layout Shift)
- **地理分佈**：訪客來源國家
- **流量來源**：Referrer 資訊
- **瀏覽器和設備**：用戶使用的瀏覽器和設備

### 效能優化

#### 1. 啟用 Cloudflare 快取

```typescript
// server/api/cached.ts
export default defineEventHandler((event) => {
  // 設定快取標頭
  event.node.res.setHeader('Cache-Control', 'public, max-age=3600, s-maxage=3600')

  return {
    message: 'This response is cached for 1 hour',
    timestamp: new Date().toISOString()
  }
})
```

#### 2. 使用圖片優化

```vue
<template>
  <!-- Cloudflare 自動優化圖片 -->
  <img
    src="/images/hero.jpg"
    alt="Hero"
    loading="lazy"
    width="800"
    height="600"
  />

  <!-- 或使用 Nuxt Image -->
  <NuxtImg
    src="/images/hero.jpg"
    width="800"
    height="600"
    format="webp"
    quality="80"
    loading="lazy"
  />
</template>
```

#### 3. 壓縮和最小化

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages',
    minify: true,
    compressPublicAssets: true
  }
})
```

#### 4. 使用 ISR（增量靜態再生）

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/blog/**': {
      swr: 3600, // 1 小時後重新生成
      cache: {
        maxAge: 3600,
        staleMaxAge: 86400 // 過期後 24 小時內仍可使用
      }
    }
  }
})
```

---

## 完整部署步驟

### 方法 1：Git 整合（推薦）

**步驟 1：準備代碼**

```bash
# 1. 配置 nuxt.config.ts
# 確保 preset 設為 'cloudflare-pages'

# 2. 測試構建
npm run build

# 3. 提交並推送
git add .
git commit -m "Ready for Cloudflare Pages"
git push origin main
```

**步驟 2：在 Cloudflare 建立項目**

1. 訪問 https://dash.cloudflare.com/pages
2. 點擊 "Create a project"
3. 連接 GitHub/GitLab
4. 選擇倉庫
5. 配置構建設定：
   ```
   Build command: npm run build
   Build output directory: .output/public
   ```
6. 點擊 "Save and Deploy"

**步驟 3：等待部署完成**

```
✓ Building application...
✓ Deploying to Cloudflare's global network...
✓ Success! https://my-nuxt-app.pages.dev
```

**步驟 4：配置自定義域名（可選）**

1. 點擊 "Custom domains"
2. 添加域名 `example.com`
3. 配置 DNS
4. 等待 SSL 證書生成

### 方法 2：Wrangler CLI

**步驟 1：安裝和配置**

```bash
# 1. 安裝 Wrangler
npm install -g wrangler

# 2. 登入
wrangler login

# 3. 建立 wrangler.toml
cat > wrangler.toml << EOF
name = "my-nuxt-app"
compatibility_date = "2024-11-01"
pages_build_output_dir = ".output/public"
EOF
```

**步驟 2：構建和部署**

```bash
# 1. 構建應用
npm run build

# 2. 部署到 Cloudflare Pages
wrangler pages deploy .output/public

# 3. 訪問部署的網站
# URL 會在部署完成後顯示
```

**步驟 3：設定環境變數**

```bash
# 設定公開變數
wrangler pages project set-env \
  NUXT_PUBLIC_API_BASE=https://api.example.com

# 設定 Secret
echo "your-secret-key" | wrangler pages secret put API_SECRET
```

### 持續部署

一旦設置完成，後續部署非常簡單：

```bash
# Git 整合：只需推送代碼
git add .
git commit -m "Update feature"
git push

# Cloudflare 會自動：
# ✓ 檢測推送
# ✓ 構建應用
# ✓ 部署到全球 CDN
# ✓ 更新預覽 URL
```

---

## 費用資訊

### Free 方案（免費）

```
價格: $0/月

包含：
- 無限網站
- 無限請求
- 無限流量
- 500 次構建/月
- 併發構建：1 個
- 自定義域名：100 個
- Preview 部署：無限
- Web Analytics
- DDoS 防護
```

**適合：** 個人項目、小型網站、大部分應用

### Pro 方案（專業）

```
價格: $20/月

包含：
- Free 方案的所有功能
- 5000 次構建/月
- 併發構建：5 個
- 優先支援
- 更長的構建時間
```

**適合：** 專業開發者、商業網站

### Business 方案（商業）

```
價格: 聯繫銷售

包含：
- Pro 方案的所有功能
- 無限構建
- 併發構建：20 個
- SLA 保證
- 24/7 支援
```

**適合：** 企業、關鍵業務應用

### 額外服務費用

```
Workers/Functions:
- 免費：100,000 請求/天
- 付費：$5/月（10M 請求）

KV Storage:
- 免費：100,000 讀取/天
- 付費：$0.50/月（1M 請求）

D1 Database:
- 免費：5GB 儲存，1M 行讀取/天
- 付費：視使用量而定

R2 Storage:
- 免費：10GB 儲存，1M 讀取/月
- 付費：$0.015/GB/月
```

### 成本優勢

對於大多數網站，Cloudflare Pages 完全免費：

```
範例：中型網站
- 100萬次頁面瀏覽/月
- 10GB 傳輸流量
- 50 次構建/月

在其他平台：
- Vercel: ~$20-40/月
- Netlify: ~$20-30/月
- AWS: ~$30-50/月

Cloudflare Pages: $0/月 ✓
```

---

## 疑難排解

### 問題 1：構建失敗

**錯誤：** Build failed

**常見原因和解決方案：**

```bash
# 1. Node.js 版本不匹配
# 在環境變數中設定：
NODE_VERSION=18

# 2. 依賴安裝失敗
# 刪除 package-lock.json 並重新生成
rm package-lock.json
npm install
git add package-lock.json
git commit -m "Update package-lock.json"
git push

# 3. 構建命令錯誤
# 確保 package.json 中有正確的構建命令
{
  "scripts": {
    "build": "nuxt build"
  }
}
```

### 問題 2：API Routes 404

**問題：** `/api/hello` 返回 404

**解決方案：**

```typescript
// 確保 nuxt.config.ts 使用正確的 preset
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages' // 不是 'cloudflare'
  }
})

// 確保 API 文件在正確位置
// server/api/hello.ts
export default defineEventHandler(() => {
  return { message: 'Hello' }
})
```

### 問題 3：環境變數未生效

**問題：** `process.env.MY_VAR` 讀不到值

**解決方案：**

```typescript
// ❌ 錯誤方式
const apiKey = process.env.MY_VAR

// ✅ 正確方式
const config = useRuntimeConfig()
const apiKey = config.public.myVar // 公開變數
const secret = config.mySecret // 私有變數（服務器端）

// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    mySecret: process.env.MY_SECRET,
    public: {
      myVar: process.env.NUXT_PUBLIC_MY_VAR
    }
  }
})
```

### 問題 4：靜態資源 404

**問題：** `/images/logo.png` 無法訪問

**解決方案：**

```
確保文件在 public 目錄中：
public/
  └── images/
      └── logo.png

訪問：/images/logo.png（不需要 /public 前綴）
```

### 問題 5：Functions 超時

**錯誤：** Worker exceeded CPU time limit

**解決方案：**

```typescript
// Workers 有 CPU 時間限制（免費方案：10ms，付費：50ms）
// 優化代碼，減少計算量

// ❌ 避免大量計算
export default defineEventHandler(() => {
  let result = 0
  for (let i = 0; i < 10000000; i++) {
    result += i
  }
  return { result }
})

// ✅ 使用緩存或預計算
export default defineEventHandler(async (event) => {
  const kv = event.context.cloudflare.env.MY_KV

  // 從 KV 讀取快取結果
  const cached = await kv.get('result')
  if (cached) return { result: cached }

  // 計算並緩存
  const result = expensiveCalculation()
  await kv.put('result', result, { expirationTtl: 3600 })

  return { result }
})
```

### 問題 6：CORS 錯誤

**解決方案：**

```typescript
// server/api/cors-example.ts
export default defineEventHandler((event) => {
  // 設定 CORS 標頭
  setResponseHeaders(event, {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization'
  })

  // 處理 OPTIONS 請求
  if (event.node.req.method === 'OPTIONS') {
    return ''
  }

  return { message: 'CORS enabled' }
})
```

或在 `nuxt.config.ts` 中全局設定：

```typescript
export default defineNuxtConfig({
  routeRules: {
    '/api/**': {
      cors: true,
      headers: {
        'Access-Control-Allow-Origin': '*'
      }
    }
  }
})
```

### 查看日誌

```bash
# 使用 Wrangler 查看即時日誌
wrangler pages deployment tail

# 在 Dashboard 查看：
# Pages 項目 → Functions → Logs
```

---

## 最佳實踐

### 1. 使用混合渲染

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'cloudflare-pages',
    prerender: {
      crawlLinks: true,
      routes: ['/']
    }
  },

  routeRules: {
    // 靜態頁面：預渲染
    '/': { prerender: true },
    '/about': { prerender: true },

    // 動態內容：ISR
    '/blog/**': { swr: 3600 },

    // 實時數據：SSR
    '/api/**': { ssr: true },

    // 客戶端：SPA
    '/dashboard/**': { ssr: false }
  }
})
```

### 2. 利用 Edge Storage

```typescript
// 使用 KV 緩存 API 響應
export default defineEventHandler(async (event) => {
  const kv = event.context.cloudflare.env.MY_KV
  const cacheKey = 'api:users'

  // 嘗試從快取讀取
  const cached = await kv.get(cacheKey, 'json')
  if (cached) return cached

  // 獲取新數據
  const data = await fetchUsers()

  // 緩存 1 小時
  await kv.put(cacheKey, JSON.stringify(data), {
    expirationTtl: 3600
  })

  return data
})
```

### 3. 優化圖片

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  image: {
    cloudflare: {
      baseURL: 'https://example.com'
    }
  }
})
```

```vue
<template>
  <!-- 自動優化 -->
  <NuxtImg
    src="/images/hero.jpg"
    width="800"
    height="600"
    format="webp"
    quality="80"
    loading="lazy"
  />
</template>
```

### 4. 設定適當的快取策略

```typescript
// server/api/articles.ts
export default defineEventHandler((event) => {
  setResponseHeaders(event, {
    'Cache-Control': 'public, max-age=3600, s-maxage=3600, stale-while-revalidate=86400'
  })

  return fetchArticles()
})
```

### 5. 使用 Preview 部署

每個分支和 PR 都會自動獲得獨立的預覽 URL：

```
main 分支: https://my-nuxt-app.pages.dev
feature-x 分支: https://feature-x.my-nuxt-app.pages.dev
PR #123: https://123.my-nuxt-app.pages.dev
```

---

## 總結

### Cloudflare Pages 優點

✅ **完全免費**（對大多數應用）：無限流量、無限請求
✅ **全球 CDN**：300+ 個節點，超低延遲
✅ **Edge Computing**：在全球邊緣運行代碼
✅ **簡單部署**：Git 整合，自動化 CI/CD
✅ **強大的生態系統**：KV、D1、R2 等服務
✅ **優秀的效能**：自動優化、壓縮、快取
✅ **安全性**：DDoS 防護、免費 SSL

### Cloudflare Pages 缺點

❌ Workers 有 CPU 時間限制
❌ 某些 Node.js API 不完全支援
❌ 較少的第三方整合（相比 Vercel）
❌ Dashboard 有時不夠直觀

### 推薦使用場景

1. **全球化網站** - 需要低延遲
2. **高流量網站** - 免費無限流量
3. **靜態或混合渲染** - SSG + ISR + SSR
4. **需要 Edge Computing** - Workers、KV、D1
5. **成本敏感項目** - 免費方案非常慷慨

### 與其他平台比較

| 特性 | Cloudflare Pages | Vercel | Netlify |
|------|-----------------|--------|---------|
| 免費流量 | ✅ 無限 | ⚠️ 100GB | ⚠️ 100GB |
| 構建時間 | ✅ 快速 | ✅ 快速 | ✅ 快速 |
| Edge Functions | ✅ Workers | ✅ Edge Functions | ✅ Edge Functions |
| 價格 | ✅ $0-20 | ⚠️ $0-20+ | ⚠️ $0-19+ |
| 易用性 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 下一步

- 嘗試部署你的 Nuxt.js 應用到 Cloudflare Pages
- 探索 Workers、KV、D1 等服務
- 學習 Edge Computing 的最佳實踐
- 對比不同部署平台的優劣

---

## 參考資源

- [Cloudflare Pages 文檔](https://developers.cloudflare.com/pages/)
- [Nuxt on Cloudflare](https://nuxt.com/deploy/cloudflare)
- [Wrangler 文檔](https://developers.cloudflare.com/workers/wrangler/)
- [Cloudflare Workers](https://developers.cloudflare.com/workers/)
- [Cloudflare D1](https://developers.cloudflare.com/d1/)
- [Cloudflare R2](https://developers.cloudflare.com/r2/)
- [Cloudflare KV](https://developers.cloudflare.com/kv/)
