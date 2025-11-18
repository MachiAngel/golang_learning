# 16. 部署到 Vercel

## Vercel 簡介與優勢

Vercel 是專為前端框架設計的雲端平台，由 Next.js 的創建者開發。它提供了最佳的 Nuxt.js 部署體驗。

### 主要優勢

- ✅ **零配置部署**：自動檢測 Nuxt.js 項目
- ✅ **全球 CDN**：遍布全球的邊緣網路
- ✅ **自動 HTTPS**：免費 SSL 證書
- ✅ **即時預覽**：每個 PR 都有獨立的預覽環境
- ✅ **優秀的 DX**：開發者體驗極佳
- ✅ **無服務器函數**：支援 API Routes
- ✅ **快速部署**：通常在 30 秒內完成

### 適用場景

- 個人項目和作品集網站
- 中小型商業網站
- 需要快速迭代的項目
- 靜態網站和 SSG 應用
- SSR 和混合渲染應用

---

## 前置準備

### 1. 註冊 Vercel 帳號

訪問 [https://vercel.com](https://vercel.com) 並使用以下方式註冊：
- GitHub 帳號（推薦）
- GitLab 帳號
- Bitbucket 帳號
- Email

### 2. 準備你的 Nuxt.js 項目

確保你的項目符合以下要求：

```json
// package.json
{
  "name": "my-nuxt-app",
  "version": "1.0.0",
  "scripts": {
    "build": "nuxt build",
    "dev": "nuxt dev",
    "generate": "nuxt generate",
    "preview": "nuxt preview"
  },
  "dependencies": {
    "nuxt": "^3.13.0",
    "vue": "^3.4.0"
  }
}
```

### 3. Git 倉庫準備

確保你的代碼已推送到 Git 倉庫：

```bash
# 初始化 Git（如果還沒有）
git init

# 添加 .gitignore
echo "node_modules
.nuxt
.output
.env
dist
.DS_Store" > .gitignore

# 提交代碼
git add .
git commit -m "Initial commit"

# 推送到 GitHub
git remote add origin https://github.com/your-username/your-repo.git
git push -u origin main
```

---

## 通過 Vercel CLI 部署

### 安裝 Vercel CLI

```bash
# 全局安裝
npm install -g vercel

# 或使用 pnpm
pnpm add -g vercel

# 或使用 yarn
yarn global add vercel
```

### 登入 Vercel

```bash
vercel login
```

系統會開啟瀏覽器，選擇你的登入方式（GitHub、GitLab 等）。

### 部署到生產環境

```bash
# 第一次部署（會提示一些配置問題）
vercel

# 或直接部署到生產環境
vercel --prod
```

### CLI 部署互動問答

```bash
? Set up and deploy "~/my-nuxt-app"? [Y/n] y
? Which scope do you want to deploy to? Your Username
? Link to existing project? [y/N] n
? What's your project's name? my-nuxt-app
? In which directory is your code located? ./
? Want to override the settings? [y/N] n
```

### 查看部署狀態

```bash
# 列出所有部署
vercel ls

# 查看當前項目的部署
vercel inspect

# 查看日誌
vercel logs
```

---

## 通過 Git 整合部署（推薦）

### 步驟 1：連接 Git 倉庫

1. 訪問 [Vercel Dashboard](https://vercel.com/dashboard)
2. 點擊 **"Add New Project"**
3. 選擇 **"Import Git Repository"**
4. 授權 Vercel 訪問你的 GitHub/GitLab/Bitbucket
5. 選擇你的 Nuxt.js 倉庫

### 步驟 2：配置項目

Vercel 會自動檢測到 Nuxt.js 項目並提供以下預設配置：

```yaml
Framework Preset: Nuxt.js
Build Command: nuxt build
Output Directory: .output/public
Install Command: npm install
Development Command: npm run dev
```

### 步驟 3：環境變數設定（可選）

在部署前可以設定環境變數：

```
NODE_VERSION=18
NUXT_PUBLIC_API_BASE=https://api.example.com
DATABASE_URL=postgresql://...
```

### 步驟 4：部署

點擊 **"Deploy"** 按鈕，Vercel 會：

1. Clone 你的倉庫
2. 安裝依賴
3. 執行構建命令
4. 部署到全球 CDN
5. 提供預覽 URL

部署完成後，你會看到：

```
✓ Production: https://my-nuxt-app.vercel.app
✓ Deployment Complete
```

---

## Nuxt.js 配置優化

### nuxt.config.ts 配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // Vercel 部署優化
  nitro: {
    preset: 'vercel',
    // 啟用預渲染
    prerender: {
      crawlLinks: true,
      routes: ['/', '/about', '/contact']
    }
  },

  // 環境變數
  runtimeConfig: {
    // 私有環境變數（僅服務器端可訪問）
    apiSecret: process.env.API_SECRET,

    // 公開環境變數（客戶端也可訪問）
    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE || 'https://api.example.com'
    }
  },

  // 生產環境優化
  app: {
    head: {
      charset: 'utf-8',
      viewport: 'width=device-width, initial-scale=1',
    }
  },

  // 路由優化
  routeRules: {
    // 靜態頁面
    '/': { prerender: true },
    '/about': { prerender: true },

    // ISR（增量靜態再生）
    '/blog/**': { swr: 3600 }, // 1 小時

    // SPA 模式
    '/dashboard/**': { ssr: false },

    // API 路由
    '/api/**': { cors: true }
  },

  // 啟用壓縮
  experimental: {
    payloadExtraction: true
  }
})
```

### vercel.json 配置（可選）

```json
{
  "buildCommand": "npm run build",
  "devCommand": "npm run dev",
  "installCommand": "npm install",
  "framework": "nuxtjs",
  "outputDirectory": ".output/public",
  "regions": ["hkg1", "sfo1"],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-XSS-Protection",
          "value": "1; mode=block"
        }
      ]
    }
  ],
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "/api/:path*"
    }
  ],
  "redirects": [
    {
      "source": "/old-blog/:slug",
      "destination": "/blog/:slug",
      "permanent": true
    }
  ]
}
```

---

## 環境變數設定

### 在 Vercel Dashboard 設定

1. 進入項目設定：`Settings` → `Environment Variables`
2. 添加環境變數：

```
變數名稱              值                              環境
-------------------------------------------------------------------
NODE_VERSION         18                              Production, Preview, Development
NUXT_PUBLIC_API_BASE https://api.example.com         Production
NUXT_PUBLIC_API_BASE https://api-staging.example.com Preview
DATABASE_URL         postgresql://...                Production (Encrypted)
API_SECRET           your-secret-key                 Production (Encrypted)
```

### 環境變數類型

```typescript
// .env.example
# 公開變數（客戶端可訪問）
NUXT_PUBLIC_API_BASE=https://api.example.com
NUXT_PUBLIC_SITE_URL=https://example.com

# 私有變數（僅服務器端）
DATABASE_URL=postgresql://localhost:5432/mydb
API_SECRET=your-secret-key
STRIPE_SECRET_KEY=sk_test_...

# Vercel 系統變數（自動提供）
# VERCEL=1
# VERCEL_ENV=production|preview|development
# VERCEL_URL=your-deployment-url.vercel.app
# VERCEL_GIT_COMMIT_SHA=abc123
```

### 在代碼中使用

```typescript
// composables/useApi.ts
export const useApi = () => {
  const config = useRuntimeConfig()

  const apiBase = config.public.apiBase
  const apiSecret = config.apiSecret // 僅服務器端可訪問

  return {
    apiBase,
    apiSecret
  }
}

// pages/index.vue
<script setup>
const { apiBase } = useApi()
console.log('API Base:', apiBase)
</script>
```

---

## 自定義域名

### 添加自定義域名

1. **進入項目設定**：選擇你的項目 → `Settings` → `Domains`

2. **添加域名**：
   ```
   輸入: example.com
   或: www.example.com
   ```

3. **配置 DNS**：

   **選項 A：使用 Vercel DNS（推薦）**
   ```
   將 Nameservers 改為：
   ns1.vercel-dns.com
   ns2.vercel-dns.com
   ```

   **選項 B：使用 A Record**
   ```
   Type: A
   Name: @
   Value: 76.76.21.21
   ```

   **選項 C：使用 CNAME**
   ```
   Type: CNAME
   Name: www
   Value: cname.vercel-dns.com
   ```

4. **等待 DNS 傳播**（通常 5-30 分鐘）

5. **自動 SSL**：Vercel 會自動配置 Let's Encrypt SSL 證書

### 域名重定向

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    // 重定向 www 到非 www
    'https://www.example.com/**': {
      redirect: 'https://example.com/**'
    }
  }
})
```

### 多域名配置

你可以為同一個項目添加多個域名：

```
主域名: example.com
別名: www.example.com
別名: app.example.com
```

---

## Preview 部署

### 自動 Preview 部署

每當你創建 Pull Request 時，Vercel 會自動：

1. 構建該分支的代碼
2. 部署到獨立的預覽 URL
3. 在 PR 中添加評論，包含預覽連結

```
✓ Preview: https://my-nuxt-app-git-feature-username.vercel.app
```

### Preview 部署的好處

- **獨立環境**：每個 PR 都有獨立的測試環境
- **快速測試**：在合併前測試新功能
- **團隊協作**：分享給設計師、產品經理查看
- **自動更新**：每次推送都會更新 Preview

### 配置 Preview 環境變數

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    public: {
      apiBase: process.env.VERCEL_ENV === 'production'
        ? 'https://api.example.com'
        : 'https://api-staging.example.com'
    }
  }
})
```

### 保護 Preview 部署

在 `Settings` → `Deployment Protection` 中：

- **Vercel Authentication**：需要登入 Vercel 才能訪問
- **Password Protection**：設定密碼保護
- **Trusted IPs**：只允許特定 IP 訪問

---

## 效能監控

### Vercel Analytics

```bash
# 安裝 Analytics
npm install @vercel/analytics
```

```typescript
// app.vue
<script setup>
import { inject } from '@vercel/analytics'

// 自動注入 Analytics
inject()
</script>
```

### Speed Insights

```bash
# 安裝 Speed Insights
npm install @vercel/speed-insights
```

```typescript
// app.vue
<script setup>
import { injectSpeedInsights } from '@vercel/speed-insights'

injectSpeedInsights()
</script>
```

### 監控指標

Vercel 提供以下監控數據：

- **Core Web Vitals**
  - LCP (Largest Contentful Paint)
  - FID (First Input Delay)
  - CLS (Cumulative Layout Shift)

- **其他指標**
  - TTFB (Time to First Byte)
  - FCP (First Contentful Paint)
  - 頁面訪問量

### 查看監控數據

1. 進入項目 Dashboard
2. 點擊 `Analytics` 標籤
3. 選擇時間範圍（24h、7d、30d）
4. 查看詳細報告

---

## 完整部署步驟（含截圖說明）

### 方法一：通過 Vercel Dashboard

**步驟 1：推送代碼到 GitHub**

```bash
# 確保代碼已提交
git add .
git commit -m "Ready for deployment"
git push origin main
```

**步驟 2：導入項目**

1. 訪問 https://vercel.com/new
2. 點擊 "Import Git Repository"
3. 選擇你的倉庫
4. 點擊 "Import"

**步驟 3：配置項目**

```
Project Name: my-nuxt-app
Framework Preset: Nuxt.js (自動檢測)
Root Directory: ./
Build Command: npm run build (自動檢測)
Output Directory: .output/public (自動檢測)
```

**步驟 4：設定環境變數（可選）**

點擊 "Environment Variables" 展開，添加：

```
NUXT_PUBLIC_API_BASE = https://api.example.com
```

**步驟 5：部署**

點擊 "Deploy" 按鈕，等待部署完成（約 1-2 分鐘）。

**步驟 6：訪問網站**

```
部署成功！
Production: https://my-nuxt-app.vercel.app
```

### 方法二：通過 Vercel CLI

```bash
# 1. 安裝 Vercel CLI
npm install -g vercel

# 2. 登入
vercel login

# 3. 首次部署（會提示配置）
vercel

# 回答問題：
# ? Set up and deploy? Yes
# ? Which scope? Your Username
# ? Link to existing project? No
# ? Project name? my-nuxt-app
# ? In which directory is your code? ./
# ? Override settings? No

# 4. 部署到生產環境
vercel --prod

# 5. 查看部署
vercel ls
```

### 持續部署流程

一旦設置完成，後續的部署非常簡單：

```bash
# 1. 修改代碼
# 2. 提交並推送
git add .
git commit -m "Update feature"
git push

# Vercel 會自動：
# ✓ 檢測到推送
# ✓ 開始構建
# ✓ 運行測試
# ✓ 部署到生產環境
# ✓ 發送通知
```

---

## 費用資訊

### Hobby 方案（免費）

```
價格: $0/月
- 100GB 流量/月
- 無限網站和 API
- 自動 HTTPS
- 預覽部署
- 基本 Analytics
- 社區支援
```

**適合：** 個人項目、作品集、學習項目

### Pro 方案

```
價格: $20/月
- 1TB 流量/月
- 高級 Analytics
- 密碼保護
- 優先支援
- 更快的構建時間
- 團隊協作
```

**適合：** 專業開發者、小型商業網站

### Enterprise 方案

```
價格: 聯繫銷售
- 自定義流量
- SLA 保證
- 專屬支援
- 高級安全功能
- SAML SSO
- 審計日誌
```

**適合：** 大型企業、關鍵業務應用

### 額外費用

```
超出流量: $40/TB
Serverless Function 執行時間:
  - Hobby: 100GB-Hours/月（免費）
  - Pro: 1000GB-Hours/月（免費）
  - 超出: $0.18/GB-Hour
```

---

## 疑難排解

### 常見問題 1：構建失敗

**錯誤訊息：**
```
Error: Command "npm run build" exited with 1
```

**解決方案：**

```bash
# 1. 檢查本地構建
npm run build

# 2. 檢查 Node.js 版本
# 在 package.json 中指定：
{
  "engines": {
    "node": ">=18.0.0"
  }
}

# 3. 或在 Vercel 中設定環境變數
NODE_VERSION=18
```

### 常見問題 2：環境變數未生效

**問題：** 環境變數在生產環境中無法訪問

**解決方案：**

```typescript
// ❌ 錯誤方式
const apiKey = process.env.API_KEY

// ✅ 正確方式
const config = useRuntimeConfig()
const apiKey = config.public.apiKey // 公開變數
const secret = config.apiSecret // 私有變數
```

### 常見問題 3：404 錯誤

**問題：** 刷新頁面時出現 404

**解決方案：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'vercel'
  }
})
```

確保使用 `vercel` preset 而不是 `vercel-edge` 或其他。

### 常見問題 4：API 路由無法訪問

**錯誤：** `/api/hello` 返回 404

**解決方案：**

```typescript
// server/api/hello.ts (正確的位置)
export default defineEventHandler((event) => {
  return {
    message: 'Hello World'
  }
})

// 確保 nuxt.config.ts 中：
export default defineNuxtConfig({
  nitro: {
    preset: 'vercel'
  }
})
```

### 常見問題 5：部署太慢

**問題：** 構建時間超過 5 分鐘

**解決方案：**

```typescript
// 1. 使用更快的包管理器
// package.json
{
  "scripts": {
    "build": "pnpm run build" // 使用 pnpm
  }
}

// 2. 啟用快取
// vercel.json
{
  "buildCommand": "npm run build",
  "cache": ["node_modules", ".nuxt"]
}

// 3. 減少預渲染頁面
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    prerender: {
      routes: ['/'] // 只預渲染必要頁面
    }
  }
})
```

### 常見問題 6：記憶體不足

**錯誤：** `JavaScript heap out of memory`

**解決方案：**

```json
// package.json
{
  "scripts": {
    "build": "NODE_OPTIONS='--max-old-space-size=4096' nuxt build"
  }
}
```

### 檢查部署日誌

```bash
# 使用 CLI 查看日誌
vercel logs

# 或在 Dashboard 中：
# 項目 → Deployments → 選擇部署 → Building 標籤
```

---

## 最佳實踐

### 1. 使用環境檢測

```typescript
// composables/useEnv.ts
export const useEnv = () => {
  const isProduction = process.env.VERCEL_ENV === 'production'
  const isPreview = process.env.VERCEL_ENV === 'preview'
  const isDevelopment = process.env.VERCEL_ENV === 'development'

  const deploymentUrl = process.env.VERCEL_URL
  const gitCommit = process.env.VERCEL_GIT_COMMIT_SHA

  return {
    isProduction,
    isPreview,
    isDevelopment,
    deploymentUrl,
    gitCommit
  }
}
```

### 2. 優化圖片

```vue
<template>
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

### 3. 使用 ISR

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/blog/**': {
      swr: 3600, // 1 小時後重新生成
      cache: {
        maxAge: 3600
      }
    }
  }
})
```

### 4. 設定正確的 Headers

```json
// vercel.json
{
  "headers": [
    {
      "source": "/fonts/(.*)",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=31536000, immutable"
        }
      ]
    }
  ]
}
```

### 5. 監控效能

定期檢查 Analytics 和 Speed Insights，確保：
- LCP < 2.5s
- FID < 100ms
- CLS < 0.1

---

## 總結

### Vercel 優點

✅ 部署極其簡單，幾乎零配置
✅ 對 Nuxt.js 支援完美
✅ 免費方案功能強大
✅ 自動 HTTPS 和全球 CDN
✅ Preview 部署對團隊協作很有幫助
✅ 效能監控工具完善

### Vercel 缺點

❌ 免費方案流量限制（100GB/月）
❌ 高流量網站費用較高
❌ 無法完全控制基礎設施
❌ Serverless 限制（執行時間、大小）

### 推薦使用場景

1. **個人項目和作品集** - 免費方案綽綽有餘
2. **中小型商業網站** - Pro 方案性價比高
3. **需要快速迭代的項目** - Preview 部署非常有用
4. **靜態和混合渲染應用** - 天然優勢

### 下一步

- 嘗試部署你的第一個 Nuxt.js 應用
- 探索 Vercel Analytics 和 Speed Insights
- 學習其他部署平台（下一章）
- 了解 CI/CD 最佳實踐

---

## 參考資源

- [Vercel 官方文檔](https://vercel.com/docs)
- [Nuxt.js on Vercel](https://vercel.com/docs/frameworks/nuxt)
- [Vercel CLI 文檔](https://vercel.com/docs/cli)
- [Vercel Analytics](https://vercel.com/analytics)
- [Vercel Pricing](https://vercel.com/pricing)
