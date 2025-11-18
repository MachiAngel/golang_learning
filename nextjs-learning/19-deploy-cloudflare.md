# 第 19 章：部署到 Cloudflare Pages

## 目錄
- [簡介](#簡介)
- [為什麼選擇 Cloudflare Pages](#為什麼選擇-cloudflare-pages)
- [平台特色](#平台特色)
- [定價與免費額度](#定價與免費額度)
- [支援的渲染模式](#支援的渲染模式)
- [部署步驟](#部署步驟)
- [環境變數設定](#環境變數設定)
- [使用 Cloudflare Workers](#使用-cloudflare-workers)
- [自訂網域與 SSL](#自訂網域與-ssl)
- [效能優化](#效能優化)
- [故障排除](#故障排除)

---

## 簡介

Cloudflare Pages 是 Cloudflare 提供的 Jamstack 部署平台，結合了 Cloudflare 的全球網路優勢，提供極快的內容分發和邊緣運算能力。它特別適合部署靜態網站和使用 Edge Runtime 的 Next.js 應用程式。

## 為什麼選擇 Cloudflare Pages

### 優勢

- **全球網路**：在 275+ 個城市的資料中心分發內容
- **無限頻寬**：免費方案提供無限頻寬
- **免費 SSL**：自動配置和更新 SSL 憑證
- **DDoS 防護**：內建 DDoS 攻擊防護
- **邊緣運算**：使用 Workers 在邊緣執行程式碼
- **快速建置**：並行建置，速度快
- **即時預覽**：每個分支和 PR 自動生成預覽
- **Git 整合**：支援 GitHub、GitLab

### 限制

- **不支援 Node.js Runtime**：僅支援 Edge Runtime
- **函數限制**：Worker 有執行時間和記憶體限制
- **建置時間**：免費方案每月 500 次建置
- **檔案大小**：單一檔案最大 25MB
- **專案數量**：免費方案最多 100 個專案

### 與 Vercel 比較

| 特性 | Cloudflare Pages | Vercel |
|------|-----------------|--------|
| **頻寬** | 無限制 | 100GB/月（免費） |
| **建置次數** | 500/月 | 無限制 |
| **Runtime** | Edge Only | Node.js + Edge |
| **價格** | $0 起 | $0 起 |
| **全球網路** | 275+ 城市 | 100+ 城市 |
| **DDoS 防護** | 包含 | 僅付費方案 |

## 平台特色

### 1. Edge Runtime

Cloudflare Pages 使用 V8 引擎在邊緣執行程式碼：

```javascript
// middleware.js
export const config = {
  runtime: 'edge', // 必須使用 Edge Runtime
}

export default function middleware(request) {
  const response = new Response('Hello from Cloudflare Edge!')
  response.headers.set('X-Edge-Location', request.cf.colo)
  return response
}
```

### 2. Cloudflare Workers

可以擴展為完整的 Workers 函數：

```javascript
// functions/api/hello.js
export async function onRequest(context) {
  return new Response(JSON.stringify({
    message: 'Hello from Cloudflare Pages!',
    location: context.request.cf.city,
    country: context.request.cf.country
  }), {
    headers: {
      'content-type': 'application/json',
    },
  })
}
```

### 3. KV 儲存

使用 Workers KV 儲存資料：

```javascript
// 綁定 KV namespace
export async function onRequest(context) {
  const { env } = context

  // 讀取
  const value = await env.MY_KV.get('key')

  // 寫入
  await env.MY_KV.put('key', 'value')

  return new Response(value)
}
```

### 4. Durable Objects

持久化狀態管理：

```javascript
export class Counter {
  constructor(state, env) {
    this.state = state
  }

  async fetch(request) {
    let count = await this.state.storage.get('count') || 0
    count++
    await this.state.storage.put('count', count)
    return new Response(count.toString())
  }
}
```

## 定價與免費額度

### 免費方案

- **頻寬**：無限制
- **建置次數**：500 次/月
- **並行建置**：1 個
- **專案數量**：100 個
- **成員**：無限制
- **自訂網域**：無限制
- **DDoS 防護**：包含
- **SSL 憑證**：免費
- **Workers 請求**：100,000 次/日
- **KV 讀取**：100,000 次/日
- **KV 寫入**：1,000 次/日
- **KV 儲存**：1 GB

### 付費方案 (Pro)

- **價格**：$20/月
- **建置次數**：5,000 次/月
- **並行建置**：5 個
- **Workers 請求**：1,000 萬次/月
- **優先建置**：更快的建置速度
- **進階分析**：詳細的效能指標
- **更多 KV 配額**：可擴展

### 企業方案

- **價格**：客製化
- **無限建置**
- **SLA 保證**：99.99% 可用性
- **專屬支援**：24/7
- **企業級安全**：SSO、SAML

## 支援的渲染模式

### ✅ 支援的模式

1. **靜態網站產生 (SSG)**
```javascript
// pages/posts/[id].js
export async function getStaticPaths() {
  return {
    paths: [{ params: { id: '1' } }],
    fallback: false
  }
}

export async function getStaticProps({ params }) {
  return {
    props: { id: params.id }
  }
}
```

2. **Edge Runtime API Routes**
```javascript
// pages/api/edge.js
export const config = {
  runtime: 'edge',
}

export default function handler(req) {
  return new Response('Edge API')
}
```

3. **Edge Middleware**
```javascript
// middleware.js
export const config = {
  runtime: 'edge',
}

export function middleware(request) {
  return Response.redirect(new URL('/new-path', request.url))
}
```

### ❌ 不支援的模式

1. **伺服器端渲染 (SSR) with Node.js Runtime**
```javascript
// ❌ 不支援
export async function getServerSideProps() {
  return { props: {} }
}
```

2. **增量靜態再生成 (ISR)**
```javascript
// ❌ 不支援
export async function getStaticProps() {
  return {
    props: {},
    revalidate: 60 // 不支援
  }
}
```

3. **Node.js API Routes**
```javascript
// ❌ 不支援 Node.js runtime
export default function handler(req, res) {
  res.json({ message: 'Hello' })
}
```

### 解決方案

對於需要 SSR 的專案，可以：
1. 使用靜態匯出 + 客戶端資料獲取
2. 使用 Edge Runtime 重寫 API
3. 考慮使用 Vercel 或其他支援 Node.js 的平台

## 部署步驟

### 方法一：透過 Cloudflare Dashboard（推薦初學者）

#### 步驟 1：準備專案

確保專案是靜態匯出或使用 Edge Runtime：

```javascript
// next.config.js
module.exports = {
  // 靜態匯出
  output: 'export',

  // 或使用 Edge Runtime
  // experimental: {
  //   runtime: 'edge',
  // },

  // 圖片優化（靜態匯出需要設定）
  images: {
    unoptimized: true,
  },

  // 移除尾隨斜線
  trailingSlash: true,
}
```

```json
// package.json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "export": "next build"
  }
}
```

#### 步驟 2：推送到 Git

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/your-username/your-repo.git
git push -u origin main
```

#### 步驟 3：登入 Cloudflare

1. 前往 [Cloudflare Pages](https://pages.cloudflare.com/)
2. 登入或建立 Cloudflare 帳號
3. 點擊「Create a project」

#### 步驟 4：連結 Git 儲存庫

1. 選擇 Git 提供商（GitHub 或 GitLab）
2. 授權 Cloudflare 存取
3. 選擇要部署的儲存庫

#### 步驟 5：設定建置

Cloudflare 會自動偵測 Next.js 專案，預設設定：

- **Framework preset**: Next.js (Static HTML Export)
- **Build command**: `npx @cloudflare/next-on-pages@1`
- **Build output directory**: `.vercel/output/static`

**靜態匯出設定**：
- **Build command**: `npm run build`
- **Build output directory**: `out`

**環境變數**（選填）：
- `NODE_VERSION`: `18`
- `NEXT_PUBLIC_API_URL`: `https://api.example.com`

#### 步驟 6：部署

1. 點擊「Save and Deploy」
2. 等待建置完成（通常 2-5 分鐘）
3. 建置成功後會顯示部署網址

#### 步驟 7：查看部署

部署網址格式：`https://your-project.pages.dev`

### 方法二：使用 Wrangler CLI

#### 步驟 1：安裝 Wrangler

```bash
npm install -g wrangler

# 或使用專案本地安裝
npm install --save-dev wrangler
```

#### 步驟 2：登入 Cloudflare

```bash
wrangler login
```

這會開啟瀏覽器進行身份驗證。

#### 步驟 3：建立專案

```bash
# 建置 Next.js 專案
npm run build

# 發布到 Pages
wrangler pages publish out --project-name=my-nextjs-app

# 首次部署會建立專案
```

#### 步驟 4：設定專案

建立 `wrangler.toml`：

```toml
name = "my-nextjs-app"
compatibility_date = "2024-01-01"

[build]
command = "npm run build"
cwd = "."

[build.upload]
format = "directory"
dir = "out"

[[build.upload.rules]]
type = "ESModule"
globs = ["**/*.js"]
fallthrough = true
```

#### 步驟 5：後續部署

```bash
# 建置並部署
npm run build && wrangler pages publish out
```

### 方法三：使用 @cloudflare/next-on-pages（推薦 Edge Runtime）

#### 步驟 1：安裝套件

```bash
npm install --save-dev @cloudflare/next-on-pages
```

#### 步驟 2：修改 Next.js 設定

```javascript
// next.config.js
const { setupDevPlatform } = require('@cloudflare/next-on-pages/next-dev')

if (process.env.NODE_ENV === 'development') {
  setupDevPlatform()
}

module.exports = {
  // 其他設定
}
```

#### 步驟 3：更新 package.json

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "preview": "npm run build && wrangler pages dev .vercel/output/static",
    "deploy": "npm run build && wrangler pages deploy .vercel/output/static"
  }
}
```

#### 步驟 4：建置和部署

```bash
# 本地預覽
npm run preview

# 部署到 Cloudflare Pages
npm run deploy
```

#### 步驟 5：設定 TypeScript（選填）

```typescript
// env.d.ts
/// <reference types="@cloudflare/workers-types" />

declare module 'cloudflare:test' {
  interface ProvidedEnv {
    MY_KV: KVNamespace
  }
}
```

### 方法四：GitHub Actions 自動部署

#### 步驟 1：建立 GitHub Action

```yaml
# .github/workflows/deploy.yml
name: Deploy to Cloudflare Pages

on:
  push:
    branches:
      - main
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      deployments: write

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: my-nextjs-app
          directory: out
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
```

#### 步驟 2：設定 Secrets

在 GitHub 儲存庫設定中新增：

1. `CLOUDFLARE_API_TOKEN`
   - 前往 Cloudflare Dashboard
   - My Profile → API Tokens → Create Token
   - 使用「Edit Cloudflare Workers」模板
   - 複製 Token

2. `CLOUDFLARE_ACCOUNT_ID`
   - 在 Cloudflare Dashboard 右側找到
   - 或從 URL 中取得

## 環境變數設定

### 在 Dashboard 設定

1. 前往 Pages 專案
2. 點擊「Settings」
3. 選擇「Environment variables」
4. 新增變數：
   - **Variable name**: `DATABASE_URL`
   - **Value**: `postgresql://...`
   - **Environment**: Production / Preview

### 使用 Wrangler 設定

```bash
# 設定環境變數（互動式）
wrangler pages secret put DATABASE_URL

# 列出所有變數
wrangler pages secret list --project-name=my-nextjs-app
```

### 在程式碼中使用

#### 公開環境變數（前端）

```javascript
// 必須以 NEXT_PUBLIC_ 開頭
const apiUrl = process.env.NEXT_PUBLIC_API_URL

export default function App() {
  return <div>API: {apiUrl}</div>
}
```

#### 私密環境變數（Edge Runtime）

```javascript
// pages/api/secret.js
export const config = {
  runtime: 'edge',
}

export default function handler(req, env) {
  // 在 Edge Runtime 中透過 env 參數存取
  const apiKey = env.SECRET_API_KEY
  return new Response(JSON.stringify({ success: true }))
}
```

#### 使用 Functions

```javascript
// functions/api/data.js
export async function onRequest(context) {
  const { env } = context

  // 存取環境變數
  const dbUrl = env.DATABASE_URL
  const apiKey = env.API_KEY

  return new Response(JSON.stringify({ dbUrl }))
}
```

### 環境變數範例

```bash
# 在 Cloudflare Dashboard 設定

# 資料庫
DATABASE_URL=postgresql://user:pass@host:5432/db

# API Keys
API_KEY=your-secret-key

# 公開變數（前端）
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_SITE_URL=https://example.com

# Cloudflare 特定
CF_ACCOUNT_ID=your-account-id
CF_ZONE_ID=your-zone-id
```

## 使用 Cloudflare Workers

### 建立 Workers 函數

#### 方法 1：使用 Functions 目錄

```javascript
// functions/api/hello.js
export async function onRequest(context) {
  const {
    request, // Request 物件
    env,     // 環境變數和綁定
    params,  // 動態路由參數
    waitUntil, // 延遲執行
    next,    // 下一個中介軟體
    data,    // 共享資料
  } = context

  return new Response(JSON.stringify({
    message: 'Hello from Worker!',
    method: request.method,
    url: request.url,
    headers: Object.fromEntries(request.headers),
  }), {
    headers: {
      'content-type': 'application/json',
    },
  })
}
```

#### 方法 2：動態路由

```javascript
// functions/api/users/[id].js
export async function onRequest(context) {
  const { params } = context
  const userId = params.id

  return new Response(JSON.stringify({
    userId,
    name: `User ${userId}`,
  }), {
    headers: { 'content-type': 'application/json' },
  })
}
```

#### 方法 3：HTTP 方法處理

```javascript
// functions/api/posts.js
export async function onRequestGet(context) {
  return new Response('GET request')
}

export async function onRequestPost(context) {
  const body = await context.request.json()
  return new Response(JSON.stringify(body))
}

export async function onRequestPut(context) {
  return new Response('PUT request')
}

export async function onRequestDelete(context) {
  return new Response('DELETE request')
}
```

### 使用 KV 儲存

#### 步驟 1：建立 KV Namespace

```bash
# 建立 KV namespace
wrangler kv:namespace create "MY_KV"

# 會輸出 namespace ID
```

#### 步驟 2：綁定到 Pages

在 Cloudflare Dashboard：
1. Pages 專案 → Settings → Functions
2. KV namespace bindings
3. 新增綁定：
   - **Variable name**: `MY_KV`
   - **KV namespace**: 選擇已建立的 namespace

#### 步驟 3：使用 KV

```javascript
// functions/api/kv.js
export async function onRequest(context) {
  const { env } = context

  // 寫入
  await env.MY_KV.put('key', 'value')
  await env.MY_KV.put('user:1', JSON.stringify({ name: 'John' }))

  // 讀取
  const value = await env.MY_KV.get('key')
  const user = await env.MY_KV.get('user:1', { type: 'json' })

  // 刪除
  await env.MY_KV.delete('key')

  // 列出 keys
  const list = await env.MY_KV.list({ prefix: 'user:' })

  return new Response(JSON.stringify({ value, user, list }))
}
```

#### KV 進階用法

```javascript
// 設定過期時間
await env.MY_KV.put('key', 'value', {
  expirationTtl: 3600, // 1 小時後過期
})

// 設定 metadata
await env.MY_KV.put('key', 'value', {
  metadata: { userId: 123, timestamp: Date.now() },
})

// 批次操作
const keys = ['key1', 'key2', 'key3']
const values = await Promise.all(
  keys.map(key => env.MY_KV.get(key))
)
```

### 使用 D1 資料庫

#### 步驟 1：建立 D1 資料庫

```bash
# 建立 D1 資料庫
wrangler d1 create my-database

# 輸出會包含 database_id
```

#### 步驟 2：執行 SQL 遷移

```sql
-- schema.sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name, email) VALUES ('John Doe', 'john@example.com');
```

```bash
# 執行遷移
wrangler d1 execute my-database --file=schema.sql
```

#### 步驟 3：綁定到 Pages

在 `wrangler.toml`：

```toml
[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "your-database-id"
```

#### 步驟 4：使用 D1

```javascript
// functions/api/users.js
export async function onRequest(context) {
  const { env } = context

  // 查詢
  const { results } = await env.DB.prepare(
    'SELECT * FROM users WHERE id = ?'
  ).bind(1).all()

  // 插入
  await env.DB.prepare(
    'INSERT INTO users (name, email) VALUES (?, ?)'
  ).bind('Jane', 'jane@example.com').run()

  // 更新
  await env.DB.prepare(
    'UPDATE users SET name = ? WHERE id = ?'
  ).bind('New Name', 1).run()

  // 刪除
  await env.DB.prepare(
    'DELETE FROM users WHERE id = ?'
  ).bind(1).run()

  return new Response(JSON.stringify(results))
}
```

### 使用 R2 儲存

```javascript
// functions/api/upload.js
export async function onRequestPost(context) {
  const { env, request } = context

  // 上傳檔案
  const formData = await request.formData()
  const file = formData.get('file')

  await env.MY_BUCKET.put('filename.txt', file.stream(), {
    httpMetadata: {
      contentType: file.type,
    },
  })

  return new Response('Uploaded')
}

export async function onRequestGet(context) {
  const { env } = context

  // 下載檔案
  const object = await env.MY_BUCKET.get('filename.txt')

  if (object === null) {
    return new Response('Not found', { status: 404 })
  }

  return new Response(object.body, {
    headers: {
      'Content-Type': object.httpMetadata.contentType,
    },
  })
}
```

## 自訂網域與 SSL

### 步驟 1：新增自訂網域

1. 前往 Pages 專案
2. 點擊「Custom domains」
3. 點擊「Set up a custom domain」
4. 輸入網域名稱（例如：`www.example.com`）

### 步驟 2：設定 DNS

#### 選項 A：使用 Cloudflare DNS（推薦）

如果網域已在 Cloudflare：
1. 自動新增 CNAME 記錄
2. 自動配置 SSL

```
Type: CNAME
Name: www
Target: your-project.pages.dev
Proxy status: Proxied (橘色雲)
```

#### 選項 B：使用外部 DNS

在 DNS 提供商新增：

```
Type: CNAME
Name: www
Value: your-project.pages.dev
```

然後在 Cloudflare 驗證。

### 步驟 3：設定根網域

對於根網域（`example.com`）：

1. 將網域 nameservers 指向 Cloudflare
2. 或使用 CNAME flattening（如果 DNS 提供商支援）

### 步驟 4：強制 HTTPS

在 Cloudflare Dashboard：
1. SSL/TLS → Edge Certificates
2. 啟用「Always Use HTTPS」
3. 啟用「Automatic HTTPS Rewrites」

### 步驟 5：設定重新導向

使用 `_redirects` 檔案：

```
# public/_redirects

# www 到非 www
https://www.example.com/* https://example.com/:splat 301

# HTTP 到 HTTPS（自動）
http://example.com/* https://example.com/:splat 301

# 特定路徑重新導向
/old-path /new-path 301
/blog/* /news/:splat 302
```

或使用 Workers：

```javascript
// functions/_middleware.js
export async function onRequest(context) {
  const url = new URL(context.request.url)

  // 重新導向 www 到非 www
  if (url.hostname === 'www.example.com') {
    url.hostname = 'example.com'
    return Response.redirect(url.toString(), 301)
  }

  return context.next()
}
```

## 效能優化

### 1. 靜態資源快取

```javascript
// functions/_middleware.js
export async function onRequest(context) {
  const response = await context.next()

  // 設定快取標頭
  if (context.request.url.includes('/_next/static/')) {
    response.headers.set('Cache-Control', 'public, max-age=31536000, immutable')
  }

  return response
}
```

### 2. 圖片優化

使用 Cloudflare Images：

```javascript
// 使用 Cloudflare Image Resizing
const imageUrl = 'https://example.com/image.jpg'
const optimized = `https://example.com/cdn-cgi/image/width=800,quality=85/${imageUrl}`
```

或在 Next.js 中：

```javascript
// next.config.js
module.exports = {
  images: {
    loader: 'custom',
    loaderFile: './cloudflare-image-loader.js',
  },
}
```

```javascript
// cloudflare-image-loader.js
export default function cloudflareLoader({ src, width, quality }) {
  const params = [`width=${width}`]
  if (quality) {
    params.push(`quality=${quality}`)
  }
  return `/cdn-cgi/image/${params.join(',')}/${src}`
}
```

### 3. 啟用 Brotli 壓縮

Cloudflare 自動啟用 Brotli 和 Gzip 壓縮。

### 4. 使用 Cloudflare Cache API

```javascript
// functions/api/cached.js
export async function onRequest(context) {
  const cache = caches.default
  const cacheKey = new Request(context.request.url, context.request)

  // 檢查快取
  let response = await cache.match(cacheKey)

  if (!response) {
    // 產生新回應
    response = new Response('Fresh data', {
      headers: {
        'Cache-Control': 'max-age=3600',
      },
    })

    // 儲存到快取
    context.waitUntil(cache.put(cacheKey, response.clone()))
  }

  return response
}
```

### 5. 邊緣快取策略

```javascript
// functions/_middleware.js
export async function onRequest(context) {
  const response = await context.next()

  // 設定 Cloudflare 快取規則
  response.headers.set('Cache-Control', 'public, max-age=3600, s-maxage=86400')

  // 自訂快取鍵
  const url = new URL(context.request.url)
  response.headers.set('CF-Cache-Tag', `page:${url.pathname}`)

  return response
}
```

### 6. 預載和預連線

```javascript
// pages/_document.js
import { Html, Head, Main, NextScript } from 'next/document'

export default function Document() {
  return (
    <Html>
      <Head>
        <link rel="preconnect" href="https://fonts.googleapis.com" />
        <link rel="dns-prefetch" href="https://api.example.com" />
      </Head>
      <body>
        <Main />
        <NextScript />
      </body>
    </Html>
  )
}
```

## 監控與分析

### 1. Web Analytics

啟用 Cloudflare Web Analytics：

```html
<!-- 在 _document.js 中加入 -->
<script
  defer
  src="https://static.cloudflareinsights.com/beacon.min.js"
  data-cf-beacon='{"token": "your-token"}'
></script>
```

### 2. Workers Analytics

查看 Workers 使用情況：

```bash
# 使用 Wrangler
wrangler pages deployment tail
```

### 3. 效能監控

```javascript
// 自訂效能指標
export function reportWebVitals(metric) {
  const body = JSON.stringify(metric)

  // 發送到分析服務
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/analytics', body)
  }
}
```

## 故障排除

### 常見問題

#### 1. 建置失敗

**錯誤**：`Build failed`

**解決方法**：

檢查建置日誌，常見原因：
- Node.js 版本不相容
- 缺少依賴項
- TypeScript 錯誤

```bash
# 本地測試建置
npm run build

# 設定 Node 版本
# 在 Cloudflare Dashboard → Settings → Environment variables
NODE_VERSION=18
```

#### 2. 404 錯誤

**問題**：部署後頁面返回 404

**解決方法**：

確認輸出目錄正確：

```javascript
// next.config.js
module.exports = {
  output: 'export',
  trailingSlash: true,
}
```

建立 `_redirects` 檔案：

```
# public/_redirects
/*    /index.html   200
```

或使用 Workers：

```javascript
// functions/_middleware.js
export async function onRequest(context) {
  const response = await context.next()

  if (response.status === 404) {
    return context.env.ASSETS.fetch(new Request('/404.html'))
  }

  return response
}
```

#### 3. 環境變數未生效

**問題**：環境變數讀取不到

**解決方法**：

1. 確認變數名稱正確（前端變數需要 `NEXT_PUBLIC_` 前綴）
2. 重新部署（修改環境變數後需要重新部署）
3. 檢查環境設定（Production vs Preview）

```bash
# 使用 Wrangler 檢查
wrangler pages secret list --project-name=my-nextjs-app
```

#### 4. Workers 執行逾時

**錯誤**：`CPU time limit exceeded`

**解決方法**：

免費方案限制：
- CPU 時間：10ms（首次請求）
- CPU 時間：50ms（後續請求）

優化建議：
- 減少計算密集型操作
- 使用快取
- 考慮升級到付費方案

```javascript
// 使用 Cache API 減少計算
export async function onRequest(context) {
  const cache = caches.default
  const cached = await cache.match(context.request)

  if (cached) {
    return cached
  }

  // 執行操作
  const response = await expensiveOperation()

  context.waitUntil(cache.put(context.request, response.clone()))

  return response
}
```

#### 5. KV 讀寫錯誤

**錯誤**：`KV namespace not found`

**解決方法**：

確認綁定正確：

```bash
# 列出 KV namespaces
wrangler kv:namespace list

# 在 Cloudflare Dashboard 確認綁定
# Pages → Settings → Functions → KV namespace bindings
```

#### 6. CORS 問題

**問題**：跨域請求被阻擋

**解決方法**：

```javascript
// functions/api/_middleware.js
export async function onRequest(context) {
  // 處理 OPTIONS 請求
  if (context.request.method === 'OPTIONS') {
    return new Response(null, {
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
      },
    })
  }

  // 處理其他請求
  const response = await context.next()

  // 新增 CORS 標頭
  response.headers.set('Access-Control-Allow-Origin', '*')

  return response
}
```

#### 7. 大檔案上傳失敗

**錯誤**：`Request body too large`

**限制**：
- 免費方案：100MB
- 付費方案：500MB

**解決方法**：

使用分塊上傳或 R2 直接上傳：

```javascript
// 產生預簽名 URL
export async function onRequest(context) {
  const { env } = context

  // 使用 R2 預簽名 URL
  const url = await env.MY_BUCKET.createPresignedUrl('file.jpg', {
    expiresIn: 3600,
  })

  return new Response(JSON.stringify({ uploadUrl: url }))
}
```

### 除錯技巧

#### 1. 本地開發

```bash
# 使用 Wrangler Pages 本地開發
npm run build
wrangler pages dev out

# 或使用 @cloudflare/next-on-pages
npm run preview
```

#### 2. 查看即時日誌

```bash
# 即時日誌
wrangler pages deployment tail

# 過濾日誌
wrangler pages deployment tail --format=json | grep "error"
```

#### 3. 除錯 Workers

```javascript
// 使用 console.log（會出現在日誌中）
export async function onRequest(context) {
  console.log('Request:', context.request.url)
  console.log('Headers:', Object.fromEntries(context.request.headers))

  return new Response('OK')
}
```

#### 4. 效能分析

在 Cloudflare Dashboard：
- Analytics → Performance
- 查看 Core Web Vitals
- 檢查錯誤率和回應時間

## 最佳實踐

### 1. 使用靜態匯出

對於不需要 SSR 的專案：

```javascript
// next.config.js
module.exports = {
  output: 'export',
  images: {
    unoptimized: true,
  },
}
```

### 2. 優化建置

```json
{
  "scripts": {
    "build": "next build && next export"
  }
}
```

### 3. 環境分離

使用不同的環境：
- **Production**: main 分支
- **Preview**: 其他分支和 PR

### 4. 安全性

```javascript
// 新增安全標頭
export async function onRequest(context) {
  const response = await context.next()

  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-Content-Type-Options', 'nosniff')
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')
  response.headers.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-inline'"
  )

  return response
}
```

### 5. 監控和告警

設定 Cloudflare 告警：
- 錯誤率超過閾值
- 回應時間過長
- Workers 執行失敗

## 總結

### Cloudflare Pages

✅ **優勢**：
- 無限頻寬（免費）
- 全球網路（275+ 城市）
- 內建 DDoS 防護
- 免費 SSL 憑證
- Workers 邊緣運算
- 快速建置和部署

⚠️ **限制**：
- 僅支援 Edge Runtime（不支援 Node.js）
- 不支援 SSR 和 ISR
- Workers 有 CPU 時間限制
- 建置次數限制（500/月免費）

🎯 **適用場景**：
- 靜態網站和部落格
- 使用 Edge Runtime 的應用
- 需要全球化部署
- 注重效能和安全性
- 成本敏感的專案

📝 **建議**：
- 如果專案需要 SSR/ISR，使用 Vercel 或 GCP
- 如果專案是靜態或 Edge Runtime，Cloudflare Pages 是絕佳選擇
- 善用 Workers KV、D1、R2 等服務擴展功能

下一章我們將學習如何部署到 Heroku。
