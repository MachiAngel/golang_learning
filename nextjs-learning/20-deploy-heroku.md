# 第 20 章：部署到 Heroku

## 目錄
- [簡介](#簡介)
- [Heroku 概述](#heroku-概述)
- [定價與免費額度](#定價與免費額度)
- [前置準備](#前置準備)
- [部署步驟](#部署步驟)
- [環境變數設定](#環境變數設定)
- [資料庫整合](#資料庫整合)
- [自訂網域與 SSL](#自訂網域與-ssl)
- [效能優化](#效能優化)
- [監控與日誌](#監控與日誌)
- [故障排除](#故障排除)

---

## 簡介

Heroku 是一個老牌的雲端平台即服務（PaaS），讓開發者能夠輕鬆部署、管理和擴展應用程式。雖然 Heroku 在 2022 年終止了免費方案，但對於需要完整後端支援的 Next.js 應用程式，它仍然是一個可靠的選擇。

> **重要更新**：Heroku 已於 2022 年 11 月 28 日終止免費方案。現在所有服務都需要付費。

## Heroku 概述

### 優勢

- **簡單部署**：使用 Git 推送即可部署
- **完整 Node.js 支援**：支援所有 Next.js 功能（SSR、ISR、API Routes）
- **豐富的附加元件**：資料庫、快取、監控等
- **自動擴展**：根據負載自動調整資源
- **CLI 工具**：強大的命令列工具
- **持續整合**：整合 GitHub、GitLab
- **Dyno 管理**：靈活的容器管理

### 劣勢

- **無免費方案**：最低 $5/月起
- **冷啟動**：閒置後首次請求較慢
- **地區限制**：僅美國和歐洲資料中心
- **成本較高**：相比其他平台較貴
- **路由限制**：30 秒請求逾時

### 適用場景

- 需要完整 Node.js 執行環境
- 使用 SSR 或 ISR
- 需要整合多種服務（資料庫、Redis 等）
- 團隊協作開發
- 需要簡單的部署流程

## 定價與免費額度

### ⚠️ 免費方案已終止

從 2022 年 11 月 28 日起，Heroku 不再提供免費 Dyno、免費 Postgres 和免費 Redis。

### 付費方案

#### Eco Dynos（入門）
- **價格**：$5/月（共享池，所有 Eco apps 共用 1000 Dyno 小時）
- **記憶體**：512 MB
- **睡眠模式**：30 分鐘無活動後休眠
- **適用**：個人專案、測試環境

#### Basic Dynos
- **價格**：$7/月/dyno
- **記憶體**：512 MB
- **無睡眠模式**：24/7 運行
- **適用**：小型生產環境

#### Standard Dynos
- **價格**：
  - Standard-1X: $25/月/dyno (512 MB)
  - Standard-2X: $50/月/dyno (1 GB)
- **效能指標**：包含
- **適用**：中型應用

#### Performance Dynos
- **價格**：
  - Performance-M: $250/月/dyno (2.5 GB)
  - Performance-L: $500/月/dyno (14 GB)
- **專用資源**：獨立 CPU
- **適用**：大型應用、高流量

### 資料庫定價

#### Heroku Postgres
- **Mini**: $5/月（20 GB 儲存）
- **Basic**: $9/月（64 GB 儲存）
- **Standard**: $50/月起（256 GB 儲存）
- **Premium**: $200/月起（1 TB 儲存）

#### Heroku Redis
- **Mini**: $3/月（25 MB）
- **Premium**: $15/月起（100 MB）

### 成本估算範例

**小型應用**：
- 1 個 Basic Dyno: $7/月
- Postgres Mini: $5/月
- **總計**: ~$12/月

**中型應用**：
- 2 個 Standard-1X Dynos: $50/月
- Postgres Basic: $9/月
- Redis Mini: $3/月
- **總計**: ~$62/月

## 前置準備

### 1. 註冊 Heroku 帳號

1. 前往 [Heroku 官網](https://www.heroku.com/)
2. 點擊「Sign up」建立帳號
3. 驗證電子郵件
4. 新增付款方式（必需）

### 2. 安裝 Heroku CLI

#### macOS

```bash
brew tap heroku/brew && brew install heroku
```

#### Windows

下載並安裝：https://devcenter.heroku.com/articles/heroku-cli

#### Linux

```bash
curl https://cli-assets.heroku.com/install.sh | sh
```

#### 驗證安裝

```bash
heroku --version
# heroku/8.7.1 linux-x64 node-v18.19.0
```

### 3. 登入 Heroku

```bash
heroku login

# 或使用非互動式登入
heroku login -i
```

這會開啟瀏覽器進行身份驗證。

### 4. 準備 Next.js 專案

確保專案可以正確運行：

```json
// package.json
{
  "name": "my-nextjs-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start -p $PORT"
  },
  "dependencies": {
    "next": "14.0.0",
    "react": "18.2.0",
    "react-dom": "18.2.0"
  },
  "engines": {
    "node": "18.x",
    "npm": "9.x"
  }
}
```

**重要**：
- `start` 腳本必須使用 `$PORT` 環境變數
- 必須指定 `engines` 欄位

## 部署步驟

### 方法一：使用 Heroku CLI（推薦）

#### 步驟 1：建立 Heroku 應用程式

```bash
# 在專案根目錄
heroku create my-nextjs-app

# 或指定地區
heroku create my-nextjs-app --region us

# 可用地區：
# - us (美國)
# - eu (歐洲)
```

這會：
- 建立 Heroku 應用程式
- 自動新增 `heroku` remote 到 Git

#### 步驟 2：設定 Buildpack

```bash
# 設定 Node.js buildpack
heroku buildpacks:set heroku/nodejs

# 查看 buildpack
heroku buildpacks
```

#### 步驟 3：設定環境變數

```bash
# 設定 Node 環境
heroku config:set NODE_ENV=production

# 設定其他變數
heroku config:set NEXT_PUBLIC_API_URL=https://api.example.com
```

#### 步驟 4：部署

```bash
# 確保程式碼已提交
git add .
git commit -m "Prepare for Heroku deployment"

# 推送到 Heroku
git push heroku main

# 如果你的主分支是 master
git push heroku master

# 如果要從其他分支部署
git push heroku develop:main
```

#### 步驟 5：開啟應用程式

```bash
# 在瀏覽器開啟
heroku open

# 查看應用程式 URL
heroku info
```

### 方法二：使用 GitHub 整合

#### 步驟 1：連結 GitHub

1. 前往 [Heroku Dashboard](https://dashboard.heroku.com/)
2. 點擊「New」→「Create new app」
3. 輸入 App name 和選擇 Region
4. 點擊「Create app」

#### 步驟 2：設定部署方法

1. 在 App 頁面，點擊「Deploy」標籤
2. Deployment method 選擇「GitHub」
3. 連結你的 GitHub 帳號
4. 搜尋並連結儲存庫

#### 步驟 3：啟用自動部署

1. 選擇要部署的分支（通常是 `main`）
2. 勾選「Wait for CI to pass before deploy」（選填）
3. 點擊「Enable Automatic Deploys」

#### 步驟 4：手動部署（首次）

1. 在「Manual deploy」區塊
2. 選擇分支
3. 點擊「Deploy Branch」

### 方法三：使用 Docker（進階）

#### 步驟 1：建立 Dockerfile

```dockerfile
# Dockerfile
FROM node:18-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:18-alpine AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000

CMD ["node", "server.js"]
```

#### 步驟 2：建立 heroku.yml

```yaml
# heroku.yml
build:
  docker:
    web: Dockerfile
run:
  web: node server.js
```

#### 步驟 3：設定並部署

```bash
# 設定 stack 為 container
heroku stack:set container

# 部署
git add .
git commit -m "Add Docker support"
git push heroku main
```

### 設定檔案

#### Procfile（選填）

如果需要自訂啟動命令：

```
# Procfile
web: npm start
```

或使用 PM2：

```
# Procfile
web: npm install pm2 -g && pm2 start npm --name "nextjs" -- start && pm2 logs
```

#### .gitignore

```
# .gitignore
node_modules/
.next/
.env*.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.DS_Store
```

## 環境變數設定

### 使用 Heroku CLI

```bash
# 設定單一變數
heroku config:set API_KEY=your-secret-key

# 設定多個變數
heroku config:set \
  DATABASE_URL=postgresql://... \
  REDIS_URL=redis://... \
  SECRET_KEY=xxx

# 查看所有變數
heroku config

# 查看特定變數
heroku config:get API_KEY

# 移除變數
heroku config:unset API_KEY

# 從 .env 檔案批次設定
heroku config:set $(cat .env | sed '/^#/d' | sed '/^$/d')
```

### 使用 Heroku Dashboard

1. 前往 App 頁面
2. 點擊「Settings」標籤
3. 點擊「Reveal Config Vars」
4. 新增 Key-Value 對

### 在 Next.js 中使用

#### 公開環境變數（前端）

```bash
# 必須以 NEXT_PUBLIC_ 開頭
heroku config:set NEXT_PUBLIC_API_URL=https://api.example.com
```

```javascript
// 在元件中使用
const apiUrl = process.env.NEXT_PUBLIC_API_URL

export default function App() {
  return <div>API: {apiUrl}</div>
}
```

#### 私密環境變數（後端）

```bash
# 不需要 NEXT_PUBLIC_ 前綴
heroku config:set SECRET_API_KEY=your-secret
heroku config:set DATABASE_URL=postgresql://...
```

```javascript
// pages/api/secret.js
export default function handler(req, res) {
  const apiKey = process.env.SECRET_API_KEY
  const dbUrl = process.env.DATABASE_URL

  // 使用變數
  res.json({ success: true })
}
```

#### Next.js 設定

```javascript
// next.config.js
module.exports = {
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
  },

  // 或使用 publicRuntimeConfig 和 serverRuntimeConfig
  publicRuntimeConfig: {
    apiUrl: process.env.NEXT_PUBLIC_API_URL,
  },
  serverRuntimeConfig: {
    secretKey: process.env.SECRET_KEY,
  },
}
```

### 環境變數範例

```bash
# Node.js
NODE_ENV=production
NPM_CONFIG_PRODUCTION=false  # 如果需要 devDependencies 進行建置

# Next.js
NEXT_TELEMETRY_DISABLED=1

# 資料庫
DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_URL=redis://user:pass@host:6379

# API Keys
SECRET_KEY=your-secret-key
API_KEY=your-api-key

# 公開變數
NEXT_PUBLIC_API_URL=https://api.example.com
NEXT_PUBLIC_SITE_URL=https://my-app.herokuapp.com
```

## 資料庫整合

### PostgreSQL

#### 步驟 1：新增 Postgres 附加元件

```bash
# 新增 Postgres (Mini plan - $5/月)
heroku addons:create heroku-postgresql:mini

# 查看資料庫資訊
heroku pg:info

# 查看連線資訊
heroku pg:credentials:url
```

這會自動設定 `DATABASE_URL` 環境變數。

#### 步驟 2：安裝資料庫客戶端

```bash
npm install pg
# 或
npm install @vercel/postgres
# 或
npm install prisma @prisma/client
```

#### 步驟 3：使用資料庫

**使用 pg**：

```javascript
// lib/db.js
import { Pool } from 'pg'

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: {
    rejectUnauthorized: false
  }
})

export default pool
```

```javascript
// pages/api/users.js
import pool from '../../lib/db'

export default async function handler(req, res) {
  try {
    const { rows } = await pool.query('SELECT * FROM users')
    res.json(rows)
  } catch (error) {
    res.status(500).json({ error: error.message })
  }
}
```

**使用 Prisma**：

```bash
# 安裝 Prisma
npm install prisma --save-dev
npm install @prisma/client

# 初始化
npx prisma init
```

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
}
```

```bash
# 執行遷移
heroku run npx prisma migrate deploy

# 或在本地
npx prisma migrate dev
```

```javascript
// lib/prisma.js
import { PrismaClient } from '@prisma/client'

const globalForPrisma = global

export const prisma =
  globalForPrisma.prisma || new PrismaClient()

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma
}
```

#### 步驟 4：資料庫操作

```bash
# 連線到資料庫
heroku pg:psql

# 執行 SQL 檔案
heroku pg:psql < schema.sql

# 備份資料庫
heroku pg:backups:capture

# 下載備份
heroku pg:backups:download

# 還原備份
heroku pg:backups:restore [BACKUP_ID]
```

### Redis

#### 步驟 1：新增 Redis 附加元件

```bash
# 新增 Redis (Mini plan - $3/月)
heroku addons:create heroku-redis:mini

# 查看 Redis 資訊
heroku redis:info

# 查看 Redis CLI
heroku redis:cli
```

這會自動設定 `REDIS_URL` 環境變數。

#### 步驟 2：安裝 Redis 客戶端

```bash
npm install ioredis
# 或
npm install redis
```

#### 步驟 3：使用 Redis

```javascript
// lib/redis.js
import Redis from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)

export default redis
```

```javascript
// pages/api/cache.js
import redis from '../../lib/redis'

export default async function handler(req, res) {
  // 設定快取
  await redis.set('key', 'value', 'EX', 3600) // 過期時間 1 小時

  // 讀取快取
  const value = await redis.get('key')

  // 刪除快取
  await redis.del('key')

  res.json({ value })
}
```

### MongoDB

#### 步驟 1：使用 MongoDB Atlas

Heroku 不提供官方 MongoDB 附加元件，建議使用 MongoDB Atlas：

1. 註冊 [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)
2. 建立免費叢集
3. 取得連線字串

#### 步驟 2：設定環境變數

```bash
heroku config:set MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/db
```

#### 步驟 3：使用 MongoDB

```bash
npm install mongodb
# 或
npm install mongoose
```

```javascript
// lib/mongodb.js
import { MongoClient } from 'mongodb'

const uri = process.env.MONGODB_URI
const options = {}

let client
let clientPromise

if (!process.env.MONGODB_URI) {
  throw new Error('Please add your Mongo URI to .env.local')
}

if (process.env.NODE_ENV === 'development') {
  if (!global._mongoClientPromise) {
    client = new MongoClient(uri, options)
    global._mongoClientPromise = client.connect()
  }
  clientPromise = global._mongoClientPromise
} else {
  client = new MongoClient(uri, options)
  clientPromise = client.connect()
}

export default clientPromise
```

## 自訂網域與 SSL

### 步驟 1：新增自訂網域

#### 使用 CLI

```bash
# 新增網域
heroku domains:add www.example.com

# 查看所有網域
heroku domains

# 移除網域
heroku domains:remove www.example.com
```

#### 使用 Dashboard

1. 前往 App 頁面
2. 點擊「Settings」標籤
3. 在「Domains」區塊點擊「Add domain」
4. 輸入網域名稱

### 步驟 2：設定 DNS

Heroku 會提供一個 DNS Target，例如：
```
example-app-12345.herokudns.com
```

在你的 DNS 提供商新增 CNAME 記錄：

```
Type: CNAME
Name: www
Value: example-app-12345.herokudns.com
```

對於根網域（`example.com`），需要使用支援 CNAME flattening 的 DNS 提供商，或使用 A 記錄指向：

```bash
# 使用 Heroku DNS
heroku domains:add example.com

# 取得 DNS targets
heroku domains:info example.com
```

### 步驟 3：啟用 SSL

Heroku 提供免費的自動 SSL 憑證（基於 Let's Encrypt）。

#### 自動 SSL（免費）

```bash
# 啟用自動 SSL
heroku certs:auto:enable

# 查看 SSL 狀態
heroku certs:auto

# 重新整理憑證
heroku certs:auto:refresh
```

SSL 憑證會在網域驗證完成後自動配置。

#### 自訂 SSL 憑證（需付費 Dyno）

如果你有自己的 SSL 憑證：

```bash
# 上傳憑證
heroku certs:add server.crt server.key

# 更新憑證
heroku certs:update server.crt server.key

# 查看憑證資訊
heroku certs:info
```

### 步驟 4：強制 HTTPS

在 Next.js 中強制使用 HTTPS：

```javascript
// middleware.js
import { NextResponse } from 'next/server'

export function middleware(request) {
  // 在 Heroku 上檢查協議
  const proto = request.headers.get('x-forwarded-proto')

  if (proto === 'http') {
    const url = request.nextUrl.clone()
    url.protocol = 'https:'
    return NextResponse.redirect(url)
  }

  return NextResponse.next()
}
```

或使用 `next.config.js`：

```javascript
// next.config.js
module.exports = {
  async redirects() {
    return [
      {
        source: '/:path*',
        has: [
          {
            type: 'header',
            key: 'x-forwarded-proto',
            value: 'http',
          },
        ],
        destination: 'https://:path*',
        permanent: true,
      },
    ]
  },
}
```

## 效能優化

### 1. 調整 Dyno 類型

```bash
# 升級到 Standard-1X (更好的效能)
heroku ps:type standard-1x

# 查看當前 Dyno 類型
heroku ps
```

### 2. 啟用 HTTP/2

Heroku 預設支援 HTTP/2，無需額外設定。

### 3. 使用 CDN

#### Cloudflare（推薦）

1. 註冊 [Cloudflare](https://www.cloudflare.com/)
2. 新增你的網域
3. 更新 Nameservers 到 Cloudflare
4. 在 Cloudflare 設定 CNAME 指向 Heroku

好處：
- 全球 CDN 加速
- DDoS 防護
- 快取靜態資源
- 免費 SSL

#### Heroku CDN 外掛

```bash
# 使用 Fastly 或其他 CDN 附加元件
heroku addons:create fastly:test
```

### 4. 快取策略

```javascript
// pages/api/data.js
export default function handler(req, res) {
  // 設定快取標頭
  res.setHeader('Cache-Control', 'public, s-maxage=60, stale-while-revalidate=300')

  res.json({ data: 'cached data' })
}
```

### 5. 壓縮

Next.js 預設啟用壓縮，但可以調整：

```javascript
// next.config.js
module.exports = {
  compress: true,

  async headers() {
    return [
      {
        source: '/:all*(svg|jpg|png)',
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

### 6. 擴展應用程式

#### 水平擴展（增加 Dynos）

```bash
# 增加 web dynos 數量
heroku ps:scale web=2

# 查看當前規模
heroku ps

# 設定自動擴展（需付費方案）
heroku autoscale:enable web \
  --min 2 \
  --max 5 \
  --p95 400
```

#### 垂直擴展（升級 Dyno 類型）

```bash
# 升級到更強大的 Dyno
heroku ps:type performance-m
```

### 7. 優化建置

```json
// package.json
{
  "scripts": {
    "heroku-postbuild": "npm run build"
  }
}
```

清理快取：

```bash
# 清除建置快取
heroku plugins:install heroku-repo
heroku repo:purge_cache -a my-nextjs-app
```

### 8. 使用 Worker Dynos

對於背景任務：

```
# Procfile
web: npm start
worker: node worker.js
```

```bash
# 啟動 worker dyno
heroku ps:scale worker=1
```

## 監控與日誌

### 查看日誌

```bash
# 即時日誌
heroku logs --tail

# 查看最近 200 行
heroku logs -n 200

# 過濾特定來源
heroku logs --source app

# 過濾特定 dyno
heroku logs --dyno web.1

# 使用時間範圍
heroku logs --since 1h
```

### 日誌管理附加元件

#### Papertrail（推薦）

```bash
# 新增 Papertrail
heroku addons:create papertrail:chopper

# 開啟 Papertrail 控制台
heroku addons:open papertrail
```

#### Logentries

```bash
heroku addons:create logentries:le_tryit
heroku addons:open logentries
```

### 效能監控

#### Heroku Metrics（內建）

```bash
# 查看效能指標
heroku metrics web

# 在 Dashboard 查看
# App → Metrics
```

可查看：
- 回應時間
- 記憶體使用
- Dyno 負載

#### New Relic（進階）

```bash
# 新增 New Relic
heroku addons:create newrelic:wayne

# 安裝 New Relic agent
npm install newrelic
```

```javascript
// 在應用程式入口引入
// server.js 或 pages/_app.js
if (process.env.NODE_ENV === 'production') {
  require('newrelic')
}
```

#### Scout APM

```bash
heroku addons:create scout:chair
npm install @scout_apm/scout-apm
```

### 錯誤追蹤

#### Sentry

```bash
npm install @sentry/nextjs
npx @sentry/wizard -i nextjs
```

```javascript
// sentry.client.config.js
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0,
})
```

```bash
# 設定環境變數
heroku config:set NEXT_PUBLIC_SENTRY_DSN=https://...
```

### 健康檢查

```javascript
// pages/api/health.js
export default function handler(req, res) {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  })
}
```

設定外部監控（例如 UptimeRobot）定期檢查。

### 自訂指標

```javascript
// lib/metrics.js
export function trackMetric(name, value) {
  // 發送到分析服務
  console.log(`[METRIC] ${name}: ${value}`)

  // 可以整合 Datadog、CloudWatch 等
}
```

## 故障排除

### 常見問題

#### 1. 應用程式啟動失敗

**錯誤**：`Application error` 或 `Error R10 (Boot timeout)`

**原因**：
- 應用程式未在 60 秒內綁定端口
- `package.json` 中缺少 `start` 腳本
- 端口設定錯誤

**解決方法**：

確保使用 `$PORT` 環境變數：

```json
// package.json
{
  "scripts": {
    "start": "next start -p $PORT"
  }
}
```

或使用自訂伺服器：

```javascript
// server.js
const { createServer } = require('http')
const { parse } = require('url')
const next = require('next')

const dev = process.env.NODE_ENV !== 'production'
const app = next({ dev })
const handle = app.getRequestHandler()

const PORT = process.env.PORT || 3000

app.prepare().then(() => {
  createServer((req, res) => {
    const parsedUrl = parse(req.url, true)
    handle(req, res, parsedUrl)
  }).listen(PORT, (err) => {
    if (err) throw err
    console.log(`> Ready on http://localhost:${PORT}`)
  })
})
```

```json
{
  "scripts": {
    "start": "node server.js"
  }
}
```

#### 2. 建置失敗

**錯誤**：`Build failed`

**常見原因**：
- Node.js 版本不相容
- 缺少依賴項
- TypeScript 錯誤
- 記憶體不足

**解決方法**：

指定 Node.js 版本：

```json
// package.json
{
  "engines": {
    "node": "18.x"
  }
}
```

增加建置記憶體：

```bash
heroku config:set NODE_OPTIONS="--max-old-space-size=4096"
```

本地測試建置：

```bash
npm run build
```

#### 3. 記憶體不足

**錯誤**：`Error R14 (Memory quota exceeded)`

**解決方法**：

```bash
# 升級 Dyno 類型
heroku ps:type standard-2x

# 優化 Next.js 建置
# next.config.js
module.exports = {
  experimental: {
    optimizeCss: true,
    optimizePackageImports: ['package-name'],
  },
}
```

#### 4. 請求逾時

**錯誤**：`Error H12 (Request timeout)`

**原因**：Heroku 路由器有 30 秒請求逾時限制

**解決方法**：

對於長時間執行的任務，使用背景任務：

```javascript
// pages/api/long-task.js
export default async function handler(req, res) {
  // 立即回應
  res.status(202).json({ message: 'Task queued' })

  // 使用 Worker dyno 或外部佇列處理
  await queue.add('long-task', req.body)
}
```

#### 5. 資料庫連線錯誤

**錯誤**：`Connection refused` 或 `SSL required`

**解決方法**：

確保啟用 SSL：

```javascript
// lib/db.js
import { Pool } from 'pg'

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: {
    rejectUnauthorized: false  // Heroku Postgres 需要
  }
})
```

檢查連線數限制：

```bash
# 查看連線資訊
heroku pg:info

# 查看當前連線
heroku pg:ps
```

#### 6. 環境變數未生效

**問題**：應用程式無法讀取環境變數

**解決方法**：

```bash
# 確認變數已設定
heroku config

# 重新啟動 dyno
heroku restart

# 檢查應用程式日誌
heroku logs --tail
```

確保前端變數使用 `NEXT_PUBLIC_` 前綴。

#### 7. 靜態資源 404

**問題**：CSS、JS、圖片無法載入

**解決方法**：

確保 `public` 目錄正確：

```
project/
├── public/
│   ├── images/
│   └── favicon.ico
├── pages/
└── package.json
```

檢查 `next.config.js`：

```javascript
module.exports = {
  // 如果部署在子路徑
  basePath: '/app',

  // 如果使用 CDN
  assetPrefix: process.env.NODE_ENV === 'production'
    ? 'https://cdn.example.com'
    : '',
}
```

#### 8. Dyno 休眠

**問題**：使用 Eco dyno，應用程式在閒置後休眠

**解決方法**：

升級到 Basic 或更高級別的 dyno：

```bash
heroku ps:type basic
```

或使用外部監控服務定期 ping（不推薦）：

```bash
# 使用 cron-job.org 或 UptimeRobot
# 每 25 分鐘訪問一次
```

### 除錯技巧

#### 1. 使用 Heroku Run

```bash
# 執行一次性命令
heroku run node --version
heroku run npm --version

# 開啟 Bash
heroku run bash

# 執行資料庫遷移
heroku run npx prisma migrate deploy
```

#### 2. 本地模擬 Heroku 環境

```bash
# 安裝 Heroku Local
# (已包含在 Heroku CLI 中)

# 執行應用程式
heroku local web

# 使用 .env 檔案
heroku local web -e .env.local
```

#### 3. 啟用除錯模式

```bash
# 設定 DEBUG 環境變數
heroku config:set DEBUG=*

# 查看詳細日誌
heroku logs --tail
```

#### 4. 檢查 Dyno 狀態

```bash
# 查看 dyno 資訊
heroku ps

# 重新啟動特定 dyno
heroku ps:restart web.1

# 重新啟動所有 dynos
heroku restart
```

#### 5. 效能分析

```bash
# 查看 dyno 負載
heroku ps:exec
# 進入後執行
top
free -m
```

## 成本優化建議

### 1. 選擇合適的 Dyno

- **開發/測試**：Eco ($5/月)
- **小型生產**：Basic ($7/月)
- **中型應用**：Standard-1X ($25/月)

### 2. 優化資料庫

```bash
# 定期清理舊資料
heroku pg:psql
# 執行 VACUUM, 刪除舊記錄等

# 監控資料庫大小
heroku pg:info
```

### 3. 使用免費附加元件

許多附加元件提供免費方案：
- Papertrail: 免費 50MB/月
- Redis: 付費（$3/月起）
- SendGrid: 免費 100 封郵件/日

### 4. 快取策略

使用 Redis 快取減少資料庫查詢：

```javascript
// lib/cache.js
import redis from './redis'

export async function getCached(key, fetcher, ttl = 3600) {
  // 嘗試從快取讀取
  const cached = await redis.get(key)
  if (cached) {
    return JSON.parse(cached)
  }

  // 快取未命中，取得新資料
  const data = await fetcher()

  // 儲存到快取
  await redis.set(key, JSON.stringify(data), 'EX', ttl)

  return data
}
```

### 5. 監控使用量

```bash
# 查看帳單
heroku billing

# 查看應用程式費用
heroku ps -a my-app
```

## 最佳實踐

### 1. 使用 Review Apps

為每個 PR 建立臨時環境：

在 `app.json` 設定：

```json
{
  "name": "my-nextjs-app",
  "description": "A Next.js application",
  "repository": "https://github.com/user/repo",
  "env": {
    "NODE_ENV": {
      "value": "production"
    },
    "NEXT_PUBLIC_API_URL": {
      "value": "https://api-staging.example.com"
    }
  },
  "addons": [
    "heroku-postgresql:mini"
  ],
  "buildpacks": [
    {
      "url": "heroku/nodejs"
    }
  ]
}
```

啟用 Review Apps：
1. 在 Dashboard 的「Deploy」標籤
2. 連結 GitHub
3. 啟用「Review Apps」
4. 勾選「Create new review apps for new pull requests automatically」

### 2. 使用 Pipeline

建立開發流程：

```bash
# 建立 pipeline
heroku pipelines:create my-app-pipeline

# 新增應用程式到不同階段
heroku pipelines:add my-app-pipeline -a my-app-staging -s staging
heroku pipelines:add my-app-pipeline -a my-app-production -s production

# 推廣到生產環境
heroku pipelines:promote -a my-app-staging
```

### 3. 資料庫備份

```bash
# 建立手動備份
heroku pg:backups:capture

# 設定自動備份（付費方案）
heroku pg:backups:schedule --at '02:00 Asia/Taipei'

# 下載備份
heroku pg:backups:download

# 還原備份
heroku pg:backups:restore [BACKUP_ID]
```

### 4. 安全性

```bash
# 使用 SSL
heroku certs:auto:enable

# 設定安全標頭
```

```javascript
// middleware.js
export function middleware(request) {
  const response = NextResponse.next()

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

### 5. 環境分離

```bash
# 建立不同環境的應用程式
heroku create my-app-dev
heroku create my-app-staging
heroku create my-app-production

# 為每個環境設定不同的 Git remote
git remote add dev https://git.heroku.com/my-app-dev.git
git remote add staging https://git.heroku.com/my-app-staging.git
git remote add production https://git.heroku.com/my-app-production.git

# 部署到不同環境
git push dev main
git push staging main
git push production main
```

## 總結

### Heroku

✅ **優勢**：
- 簡單部署流程（Git push）
- 完整 Node.js 支援（SSR、ISR、API Routes）
- 豐富的附加元件生態系
- 強大的 CLI 工具
- Pipeline 和 Review Apps

⚠️ **劣勢**：
- 無免費方案（最低 $5/月）
- 成本較高（相比其他平台）
- 30 秒請求逾時限制
- 地區選擇有限
- 冷啟動問題（Eco/Basic dynos）

💰 **成本考量**：
- **最小成本**：~$12/月（Basic dyno + Postgres Mini）
- **建議配置**：~$30-50/月（Standard dyno + 附加元件）
- **企業級**：$100+/月

🎯 **適用場景**：
- 需要完整 Node.js 執行環境
- 使用 SSR 或 ISR
- 需要多種服務整合（資料庫、Redis、監控等）
- 重視部署簡單性
- 預算充足的專案

📝 **替代方案**：
- **Vercel**：更適合 Next.js，免費方案更好
- **Render**：類似 Heroku，價格更低
- **Railway**：現代化替代方案
- **Fly.io**：全球部署，按使用量計費

---

**四個平台比較總結**：

| 平台 | 最適合 | 免費額度 | 價格 | 複雜度 |
|------|--------|---------|------|--------|
| **Vercel** | Next.js 專案 | ✅ 慷慨 | $0-20/月 | ⭐ 簡單 |
| **GCP Cloud Run** | 容器化應用 | ✅ 有 | $0-10/月 | ⭐⭐ 中等 |
| **Cloudflare Pages** | 靜態網站 | ✅ 最好 | $0/月 | ⭐⭐ 中等 |
| **Heroku** | 傳統應用 | ❌ 無 | $12+/月 | ⭐ 簡單 |

選擇建議：
- 🥇 **Next.js 首選**：Vercel
- 🥈 **成本最低**：Cloudflare Pages（靜態）或 GCP Cloud Run（動態）
- 🥉 **最簡單**：Heroku（願意付費）
- 🏆 **最彈性**：GCP（需要深度客製化）
