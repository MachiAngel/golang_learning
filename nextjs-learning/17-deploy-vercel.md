# 第 17 章：部署到 Vercel（官方推薦平台）

## 目錄
- [簡介](#簡介)
- [為什麼選擇 Vercel](#為什麼選擇-vercel)
- [平台特色](#平台特色)
- [定價與免費額度](#定價與免費額度)
- [部署步驟](#部署步驟)
- [環境變數設定](#環境變數設定)
- [進階設定](#進階設定)
- [自訂網域](#自訂網域)
- [效能優化](#效能優化)
- [故障排除](#故障排除)

---

## 簡介

Vercel 是 Next.js 的官方推薦部署平台，由 Next.js 的創建者 Vercel 公司開發和維護。它提供了最佳的 Next.js 部署體驗，包括自動化部署、邊緣網路、即時預覽等功能。

## 為什麼選擇 Vercel

### 優勢
- **零配置部署**：針對 Next.js 優化，無需額外設定
- **自動優化**：自動進行程式碼分割、圖片優化、字體優化
- **全球 CDN**：內容自動分發到全球邊緣節點
- **即時預覽**：每個 Pull Request 自動生成預覽環境
- **一鍵回滾**：輕鬆回滾到先前的部署版本
- **邊緣函數**：支援 Edge Runtime 和 Middleware
- **分析工具**：內建網站分析和效能監控
- **持續整合**：自動化 CI/CD 流程

### 限制
- **免費方案限制**：
  - 單一專案每月 100GB 頻寬
  - 無法使用進階分析功能
  - 商業使用需付費方案
- **地區限制**：某些功能僅在特定區域可用
- **供應商鎖定**：深度整合可能導致遷移困難

## 平台特色

### 1. Serverless Functions
```javascript
// pages/api/hello.js
export default function handler(req, res) {
  res.status(200).json({ message: 'Hello from Vercel!' })
}
```

### 2. Edge Middleware
```javascript
// middleware.js
import { NextResponse } from 'next/server'

export function middleware(request) {
  // 在邊緣節點執行
  const response = NextResponse.next()
  response.headers.set('x-custom-header', 'vercel-edge')
  return response
}
```

### 3. 增量靜態再生成 (ISR)
```javascript
// pages/posts/[id].js
export async function getStaticProps({ params }) {
  const post = await fetchPost(params.id)

  return {
    props: { post },
    revalidate: 60 // 每 60 秒重新生成
  }
}
```

## 定價與免費額度

### 免費方案 (Hobby)
- **價格**：$0/月
- **頻寬**：100GB/月
- **部署次數**：無限制
- **團隊成員**：1 人
- **專案數量**：無限制
- **建置時間**：6,000 分鐘/月
- **Serverless Functions**：100GB-小時
- **Edge Functions**：500,000 次執行/月
- **適用對象**：個人專案、學習、測試

### Pro 方案
- **價格**：$20/月
- **頻寬**：1TB/月
- **建置時間**：24,000 分鐘/月
- **團隊成員**：無限制
- **進階分析**：包含
- **優先支援**：包含
- **適用對象**：專業開發者、小型企業

### Enterprise 方案
- **價格**：客製化
- **專屬支援**：24/7 支援
- **SLA 保證**：99.99% 可用性
- **企業級安全**：SSO、稽核日誌
- **適用對象**：大型企業

## 部署步驟

### 方法一：透過 Vercel 網站部署（推薦初學者）

#### 步驟 1：準備專案

確保你的 Next.js 專案已經推送到 Git 儲存庫（GitHub、GitLab 或 Bitbucket）。

```bash
# 初始化 Git（如果還沒有）
git init
git add .
git commit -m "Initial commit"

# 推送到 GitHub
git remote add origin https://github.com/your-username/your-repo.git
git push -u origin main
```

#### 步驟 2：註冊 Vercel 帳號

1. 前往 [Vercel 官網](https://vercel.com)
2. 點擊「Sign Up」
3. 選擇使用 GitHub、GitLab 或 Bitbucket 登入
4. 授權 Vercel 存取你的儲存庫

#### 步驟 3：匯入專案

1. 登入後，點擊「Add New...」→「Project」
2. 選擇你要部署的儲存庫
3. 點擊「Import」

#### 步驟 4：設定專案

Vercel 會自動偵測 Next.js 專案並提供預設設定：

- **Framework Preset**：Next.js（自動偵測）
- **Root Directory**：./（如果專案在子目錄，需調整）
- **Build Command**：`next build`（預設）
- **Output Directory**：`.next`（預設）
- **Install Command**：`npm install`（預設）

如需自訂，可以在這裡修改。

#### 步驟 5：設定環境變數（選填）

如果專案需要環境變數：

1. 展開「Environment Variables」區塊
2. 新增變數，例如：
   - Name: `DATABASE_URL`
   - Value: `postgresql://...`
   - Environment: Production（或 Preview、Development）

#### 步驟 6：部署

1. 點擊「Deploy」按鈕
2. 等待部署完成（通常 1-3 分鐘）
3. 部署成功後，你會看到：
   - 預覽截圖
   - 部署網址（例如：`your-app.vercel.app`）
   - 部署日誌

### 方法二：使用 Vercel CLI（推薦進階用戶）

#### 步驟 1：安裝 Vercel CLI

```bash
# 使用 npm
npm install -g vercel

# 或使用 yarn
yarn global add vercel

# 或使用 pnpm
pnpm add -g vercel
```

#### 步驟 2：登入 Vercel

```bash
vercel login
```

這會開啟瀏覽器進行身份驗證。

#### 步驟 3：部署專案

在專案根目錄執行：

```bash
# 第一次部署（互動式設定）
vercel

# 系統會詢問：
# ? Set up and deploy "~/your-project"? [Y/n] y
# ? Which scope do you want to deploy to? Your Name
# ? Link to existing project? [y/N] n
# ? What's your project's name? my-nextjs-app
# ? In which directory is your code located? ./
```

#### 步驟 4：部署到生產環境

```bash
# 部署到生產環境
vercel --prod

# 或簡寫
vercel -p
```

#### 步驟 5：查看部署資訊

```bash
# 查看專案資訊
vercel ls

# 查看部署狀態
vercel inspect [deployment-url]

# 查看日誌
vercel logs [deployment-url]
```

### 方法三：GitHub 自動部署（推薦團隊協作）

一旦在 Vercel 上匯入專案，就會自動設定：

1. **自動部署**：
   - Push 到主分支 → 自動部署到生產環境
   - Push 到其他分支 → 自動部署到預覽環境

2. **Pull Request 預覽**：
   - 每個 PR 自動生成獨立預覽環境
   - 在 PR 中直接看到預覽連結
   - 每次提交都會更新預覽

3. **設定工作流程**：
```yaml
# .github/workflows/vercel-preview.yml（選填）
name: Vercel Preview
on:
  pull_request:
    branches: [main]

jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.ORG_ID }}
          vercel-project-id: ${{ secrets.PROJECT_ID }}
```

## 環境變數設定

### 在網站介面設定

1. 前往專案儀表板
2. 點擊「Settings」標籤
3. 選擇「Environment Variables」
4. 新增變數：
   - **Name**：變數名稱
   - **Value**：變數值
   - **Environment**：選擇環境（Production、Preview、Development）

### 使用 Vercel CLI 設定

```bash
# 新增環境變數
vercel env add DATABASE_URL

# 系統會詢問要套用到哪些環境
# ? What's the value of DATABASE_URL? postgresql://...
# ? Add DATABASE_URL to which Environments? [Production, Preview, Development]

# 列出所有環境變數
vercel env ls

# 拉取環境變數到本地（.env.local）
vercel env pull

# 移除環境變數
vercel env rm DATABASE_URL
```

### 環境變數類型

#### 1. 系統環境變數（自動提供）

Vercel 自動提供以下環境變數：

```javascript
process.env.VERCEL // "1"
process.env.VERCEL_ENV // "production" | "preview" | "development"
process.env.VERCEL_URL // 部署的 URL
process.env.VERCEL_GIT_COMMIT_SHA // Git commit SHA
process.env.VERCEL_GIT_COMMIT_REF // Git branch
```

#### 2. 公開環境變數（前端可存取）

變數名稱必須以 `NEXT_PUBLIC_` 開頭：

```bash
# 在 Vercel 設定
NEXT_PUBLIC_API_URL=https://api.example.com
```

```javascript
// 在 React 元件中使用
export default function App() {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL
  return <div>API URL: {apiUrl}</div>
}
```

#### 3. 私密環境變數（後端專用）

不需要 `NEXT_PUBLIC_` 前綴，僅在伺服器端可用：

```javascript
// pages/api/secret.js
export default function handler(req, res) {
  const apiKey = process.env.SECRET_API_KEY // 僅伺服器端可存取
  res.json({ message: 'Secret accessed' })
}
```

### 從檔案批次匯入

```bash
# 從 .env 檔案匯入（本地開發）
vercel env pull

# 這會建立 .env.local 檔案
```

在專案中使用：

```javascript
// next.config.js
module.exports = {
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
  },
}
```

## 進階設定

### vercel.json 設定檔

在專案根目錄建立 `vercel.json`：

```json
{
  "version": 2,
  "buildCommand": "npm run build",
  "devCommand": "npm run dev",
  "installCommand": "npm install",
  "framework": "nextjs",
  "regions": ["hnd1", "sfo1"],
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        {
          "key": "Access-Control-Allow-Origin",
          "value": "*"
        },
        {
          "key": "Access-Control-Allow-Methods",
          "value": "GET, POST, PUT, DELETE, OPTIONS"
        }
      ]
    }
  ],
  "redirects": [
    {
      "source": "/old-path",
      "destination": "/new-path",
      "permanent": true
    },
    {
      "source": "/blog/:slug",
      "destination": "/news/:slug",
      "permanent": false
    }
  ],
  "rewrites": [
    {
      "source": "/api/:path*",
      "destination": "https://api.example.com/:path*"
    }
  ],
  "functions": {
    "api/**/*.js": {
      "memory": 1024,
      "maxDuration": 10
    }
  },
  "crons": [
    {
      "path": "/api/cron",
      "schedule": "0 0 * * *"
    }
  ]
}
```

### 自訂建置設定

```json
{
  "build": {
    "env": {
      "NODE_ENV": "production",
      "NEXT_TELEMETRY_DISABLED": "1"
    }
  },
  "buildCommand": "npm run build && npm run post-build",
  "outputDirectory": ".next"
}
```

### Monorepo 設定

```json
{
  "version": 2,
  "buildCommand": "cd ../.. && npx turbo run build --filter=web",
  "outputDirectory": ".next",
  "installCommand": "npm install --prefix=../.."
}
```

### 區域設定

指定部署的地理區域：

```json
{
  "regions": ["hnd1", "sfo1", "iad1"]
}
```

可用區域：
- `hnd1` - 東京
- `sfo1` - 舊金山
- `iad1` - 華盛頓特區
- `sin1` - 新加坡
- `fra1` - 法蘭克福

## 自訂網域

### 步驟 1：新增網域

1. 前往專案儀表板
2. 點擊「Settings」→「Domains」
3. 輸入你的網域名稱（例如：`example.com`）
4. 點擊「Add」

### 步驟 2：設定 DNS

#### 選項 A：使用 Vercel DNS（推薦）

如果你的網域註冊商支援，可以將 DNS 管理轉移到 Vercel：

1. 在 Vercel 中選擇「Use Vercel DNS」
2. 在網域註冊商處更新 Nameservers 為：
   ```
   ns1.vercel-dns.com
   ns2.vercel-dns.com
   ```
3. 等待 DNS 傳播（通常 24-48 小時）

#### 選項 B：使用自己的 DNS

在你的 DNS 提供商新增以下記錄：

```
# A 記錄
Type: A
Name: @
Value: 76.76.21.21

# CNAME 記錄（www 子網域）
Type: CNAME
Name: www
Value: cname.vercel-dns.com
```

### 步驟 3：驗證網域

1. DNS 設定完成後，回到 Vercel
2. 點擊「Refresh」驗證 DNS 設定
3. 驗證成功後，SSL 憑證會自動配置

### 步驟 4：設定重新導向

```json
// vercel.json
{
  "redirects": [
    {
      "source": "/:path*",
      "destination": "https://www.example.com/:path*",
      "permanent": true,
      "has": [
        {
          "type": "host",
          "value": "example.com"
        }
      ]
    }
  ]
}
```

### 子網域設定

```bash
# 使用 Vercel CLI 新增子網域
vercel domains add api.example.com
```

在 Vercel 儀表板中：
1. 新增子網域（例如：`api.example.com`）
2. 在 DNS 新增 CNAME 記錄：
   ```
   Type: CNAME
   Name: api
   Value: cname.vercel-dns.com
   ```

## 效能優化

### 1. 圖片優化

使用 Next.js Image 元件：

```javascript
import Image from 'next/image'

export default function MyImage() {
  return (
    <Image
      src="/photo.jpg"
      alt="Photo"
      width={500}
      height={300}
      quality={75}
      placeholder="blur"
      loading="lazy"
    />
  )
}
```

### 2. 啟用快取策略

```javascript
// next.config.js
module.exports = {
  async headers() {
    return [
      {
        source: '/:all*(svg|jpg|png|webp)',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
    ]
  },
}
```

### 3. 使用 Edge Functions

```javascript
// middleware.js
export const config = {
  matcher: '/api/edge/:path*',
}

export default function middleware(request) {
  // 在邊緣節點執行，延遲更低
  return new Response(JSON.stringify({ message: 'Hello from Edge!' }), {
    headers: { 'content-type': 'application/json' },
  })
}
```

### 4. 啟用壓縮

Vercel 自動啟用 Brotli 和 Gzip 壓縮，無需額外設定。

### 5. 分析效能

在 Vercel 儀表板查看：
- **Analytics**：頁面瀏覽、訪客統計
- **Speed Insights**：Core Web Vitals 指標
- **Real Experience Score**：真實用戶體驗評分

啟用 Web Vitals 報告：

```javascript
// pages/_app.js
export function reportWebVitals(metric) {
  console.log(metric)
  // 或發送到分析服務
  if (metric.label === 'web-vital') {
    // 發送到 Analytics
  }
}
```

## 故障排除

### 常見問題

#### 1. 建置失敗

**問題**：`Error: Build failed`

**解決方法**：

```bash
# 檢查建置日誌
vercel logs [deployment-url]

# 本地測試建置
npm run build

# 清除快取重新建置
vercel --force
```

**常見原因**：
- TypeScript 型別錯誤
- 缺少環境變數
- 依賴項版本衝突
- Node.js 版本不相容

#### 2. 環境變數未生效

**問題**：環境變數在應用程式中讀取不到

**解決方法**：

1. 確認變數名稱正確（區分大小寫）
2. 前端變數必須以 `NEXT_PUBLIC_` 開頭
3. 重新部署專案（修改環境變數後需要重新部署）

```bash
# 拉取最新環境變數
vercel env pull

# 重新部署
vercel --prod
```

#### 3. 404 錯誤

**問題**：部署後某些路徑返回 404

**解決方法**：

檢查 `vercel.json` 的重寫規則：

```json
{
  "rewrites": [
    {
      "source": "/:path*",
      "destination": "/index.html"
    }
  ]
}
```

或使用 Next.js 的 `trailingSlash` 設定：

```javascript
// next.config.js
module.exports = {
  trailingSlash: true,
}
```

#### 4. 函數超時

**問題**：`Error: FUNCTION_INVOCATION_TIMEOUT`

**解決方法**：

1. 優化函數執行時間
2. 增加函數超時設定（Pro 方案以上）

```json
// vercel.json
{
  "functions": {
    "api/**/*.js": {
      "maxDuration": 60
    }
  }
}
```

免費方案限制：
- Serverless Functions: 10 秒
- Edge Functions: 1 秒（無法調整）

#### 5. 記憶體不足

**問題**：`Error: Memory limit exceeded`

**解決方法**：

```json
// vercel.json
{
  "functions": {
    "api/**/*.js": {
      "memory": 3008
    }
  }
}
```

可用記憶體選項：128, 256, 512, 1024, 3008 MB

#### 6. CORS 錯誤

**問題**：API 請求被 CORS 政策阻擋

**解決方法**：

```javascript
// pages/api/data.js
export default function handler(req, res) {
  res.setHeader('Access-Control-Allow-Origin', '*')
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE')
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type')

  if (req.method === 'OPTIONS') {
    res.status(200).end()
    return
  }

  res.json({ data: 'Hello' })
}
```

或使用 `vercel.json`：

```json
{
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Access-Control-Allow-Origin", "value": "*" },
        { "key": "Access-Control-Allow-Methods", "value": "GET,POST,PUT,DELETE" }
      ]
    }
  ]
}
```

#### 7. 圖片優化失敗

**問題**：圖片無法載入或優化失敗

**解決方法**：

確認 `next.config.js` 設定：

```javascript
module.exports = {
  images: {
    domains: ['example.com', 'cdn.example.com'],
    formats: ['image/avif', 'image/webp'],
    minimumCacheTTL: 60,
  },
}
```

#### 8. 部署卡住

**問題**：部署進度停滯不前

**解決方法**：

```bash
# 取消當前部署
vercel cancel

# 清除快取重新部署
vercel --force

# 檢查建置命令是否有問題
vercel build
```

### 除錯技巧

#### 1. 啟用詳細日誌

```bash
# 本地開發
DEBUG=* npm run dev

# 部署時查看詳細日誌
vercel --debug
```

#### 2. 使用 Vercel Logs

```bash
# 即時查看日誌
vercel logs --follow

# 查看特定部署的日誌
vercel logs [deployment-url]

# 過濾日誌
vercel logs --since 1h
```

#### 3. 本地模擬生產環境

```bash
# 建置生產版本
npm run build

# 啟動生產伺服器
npm start

# 或使用 Vercel CLI
vercel dev
```

#### 4. 檢查部署狀態

```bash
# 列出所有部署
vercel ls

# 查看部署詳情
vercel inspect [deployment-url]

# 檢查專案設定
vercel project ls
```

### 效能監控

#### 1. 使用 Vercel Analytics

在專案中啟用：

```javascript
// pages/_app.js
import { Analytics } from '@vercel/analytics/react'

function MyApp({ Component, pageProps }) {
  return (
    <>
      <Component {...pageProps} />
      <Analytics />
    </>
  )
}

export default MyApp
```

#### 2. Speed Insights

```bash
npm install @vercel/speed-insights
```

```javascript
// pages/_app.js
import { SpeedInsights } from '@vercel/speed-insights/next'

function MyApp({ Component, pageProps }) {
  return (
    <>
      <Component {...pageProps} />
      <SpeedInsights />
    </>
  )
}
```

## 最佳實踐

### 1. 版本控制

```bash
# 使用語義化版本標籤
git tag v1.0.0
git push origin v1.0.0

# 在 Vercel 中可以基於標籤部署
```

### 2. 環境分離

- **Production**：主分支（main/master）
- **Preview**：功能分支、Pull Requests
- **Development**：本地開發環境

```javascript
// lib/config.js
export const config = {
  apiUrl: process.env.VERCEL_ENV === 'production'
    ? 'https://api.example.com'
    : 'https://api-staging.example.com'
}
```

### 3. 安全性

```javascript
// 使用環境變數儲存敏感資訊
const apiKey = process.env.SECRET_API_KEY

// 永不提交到版本控制
// .gitignore
.env*.local
.vercel
```

### 4. 效能優化

```javascript
// next.config.js
module.exports = {
  // 壓縮
  compress: true,

  // 生產環境移除 console
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },

  // SWC 編譯
  swcMinify: true,
}
```

### 5. SEO 優化

```javascript
// pages/_app.js
import Head from 'next/head'

function MyApp({ Component, pageProps }) {
  return (
    <>
      <Head>
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <link rel="icon" href="/favicon.ico" />
      </Head>
      <Component {...pageProps} />
    </>
  )
}
```

## 總結

Vercel 是部署 Next.js 應用程式的最佳選擇，提供：

✅ **優勢**：
- 零配置、自動優化
- 全球 CDN、邊緣函數
- 免費方案已足夠個人專案使用
- 完善的開發者體驗

⚠️ **注意事項**：
- 免費方案有頻寬限制
- 商業使用需付費方案
- 某些進階功能需要升級

🎯 **適用場景**：
- Next.js 專案（首選）
- 需要全球化部署的應用
- 需要自動化 CI/CD 的專案
- 重視開發者體驗的團隊

下一章我們將學習如何部署到 Google Cloud Platform。
