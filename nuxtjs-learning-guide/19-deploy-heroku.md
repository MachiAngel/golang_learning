# 19. 部署到 Heroku

## Heroku 簡介

Heroku 是一個歷史悠久的雲端 PaaS（Platform as a Service）平台，以其簡單易用而聞名。雖然在 2022 年取消了免費方案，但對於需要快速部署和管理應用的團隊來說，仍然是一個可靠的選擇。

### 主要優勢

- ✅ **簡單易用**：對開發者友好的介面
- ✅ **Git 整合**：`git push` 即可部署
- ✅ **豐富的 Add-ons**：資料庫、快取、監控等
- ✅ **自動擴展**：根據負載自動調整資源
- ✅ **完善的文檔**：豐富的學習資源
- ✅ **企業級支援**：適合商業應用
- ✅ **多語言支援**：支援多種程式語言

### Heroku 的變化

**重要提示：** 從 2022 年 11 月 28 日起，Heroku 停止提供免費方案。

```
之前：
- Free: $0/月（512MB RAM，550-1000 dyno hours）
- Hobby: $7/月

現在：
- Eco: $5/月（適合個人項目，可以休眠）
- Basic: $7/月（不休眠）
- Standard: $25-50/月
- Performance: $250-500/月
```

### 適用場景

- 需要資料庫的全棧應用
- 需要 Add-ons 整合的項目
- 企業級應用（有預算）
- 原型開發和 MVP
- 已熟悉 Heroku 的團隊

---

## 前置準備

### 1. 註冊 Heroku 帳號

訪問 [https://signup.heroku.com/](https://signup.heroku.com/) 註冊帳號。

### 2. 安裝 Heroku CLI

#### macOS

```bash
# 使用 Homebrew
brew tap heroku/brew && brew install heroku

# 驗證安裝
heroku --version
```

#### Ubuntu/Debian

```bash
# 使用 snap
sudo snap install --classic heroku

# 或使用腳本
curl https://cli-assets.heroku.com/install.sh | sh

# 驗證安裝
heroku --version
```

#### Windows

```bash
# 下載並安裝
# https://devcenter.heroku.com/articles/heroku-cli

# 或使用 npm
npm install -g heroku
```

### 3. 登入 Heroku

```bash
# 登入（會開啟瀏覽器）
heroku login

# 或使用互動式登入
heroku login -i

# 驗證登入狀態
heroku auth:whoami
```

### 4. 準備 Nuxt.js 項目

確保你的項目已經初始化為 Git 倉庫：

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
npm-debug.log
yarn-error.log
EOF
```

---

## Procfile 建立

Procfile 告訴 Heroku 如何運行你的應用。

### 基本 Procfile

在項目根目錄建立 `Procfile`（注意：沒有副檔名）：

```bash
# Procfile
web: node .output/server/index.mjs
```

### 完整的 Procfile（包含 worker）

```bash
# Procfile
web: node .output/server/index.mjs
worker: node worker.js
release: npm run db:migrate
```

說明：
- **web**：Web 服務器（必須）
- **worker**：背景任務（可選）
- **release**：部署前執行的命令（可選）

### 指定端口

Heroku 會自動分配端口，通過 `PORT` 環境變數傳遞：

```javascript
// Nuxt 會自動使用 process.env.PORT
// 無需額外配置
```

---

## Package.json 配置

### 完整的 package.json

```json
{
  "name": "my-nuxt-app",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "build": "nuxt build",
    "dev": "nuxt dev",
    "generate": "nuxt generate",
    "preview": "nuxt preview",
    "start": "node .output/server/index.mjs",
    "postinstall": "nuxt prepare"
  },
  "dependencies": {
    "nuxt": "^3.13.0",
    "vue": "^3.4.0"
  },
  "devDependencies": {
    "@nuxt/devtools": "latest"
  },
  "engines": {
    "node": "18.x",
    "npm": "9.x"
  }
}
```

**重要欄位說明：**

- **`engines`**：指定 Node.js 和 npm 版本
- **`start`**：Heroku 會執行此命令啟動應用
- **`postinstall`**：安裝後自動準備 Nuxt

---

## Nuxt.js 配置

### nuxt.config.ts

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 使用 node-server preset
  nitro: {
    preset: 'node-server'
  },

  // 運行時配置
  runtimeConfig: {
    // 私有環境變數（僅服務器端可訪問）
    apiSecret: process.env.API_SECRET,
    databaseUrl: process.env.DATABASE_URL,

    public: {
      // 公開環境變數（客戶端也可訪問）
      apiBase: process.env.NUXT_PUBLIC_API_BASE || 'https://api.example.com',
      appUrl: process.env.NUXT_PUBLIC_APP_URL || 'https://my-app.herokuapp.com'
    }
  },

  // 應用配置
  app: {
    head: {
      charset: 'utf-8',
      viewport: 'width=device-width, initial-scale=1',
      title: 'My Nuxt App'
    }
  },

  // 開發工具（生產環境會自動禁用）
  devtools: { enabled: true }
})
```

---

## Heroku CLI 設定

### 建立 Heroku 應用

```bash
# 建立新應用
heroku create my-nuxt-app

# 或指定區域
heroku create my-nuxt-app --region us

# 可用區域：
# - us (美國)
# - eu (歐洲)

# 查看應用資訊
heroku apps:info my-nuxt-app
```

建立成功後會顯示：

```
Creating app... done, ⬢ my-nuxt-app
https://my-nuxt-app.herokuapp.com/ | https://git.heroku.com/my-nuxt-app.git
```

### 設定 Buildpack

Heroku 會自動檢測 Node.js 項目，但你也可以手動設定：

```bash
# 設定 Node.js buildpack
heroku buildpacks:set heroku/nodejs

# 查看 buildpack
heroku buildpacks

# 添加多個 buildpacks（如果需要）
heroku buildpacks:add --index 1 heroku/nodejs
```

### 查看應用狀態

```bash
# 查看應用資訊
heroku apps:info

# 查看 dynos 狀態
heroku ps

# 查看最近的部署
heroku releases

# 查看配置
heroku config
```

---

## 通過 Git 部署

### 首次部署

```bash
# 1. 確保所有變更已提交
git add .
git commit -m "Initial commit for Heroku"

# 2. 添加 Heroku remote（如果還沒有）
heroku git:remote -a my-nuxt-app

# 3. 推送到 Heroku
git push heroku main

# 如果你的主分支是 master
git push heroku master
```

部署過程：

```
Counting objects: 100, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (50/50), done.
Writing objects: 100% (100/100), 1.23 MiB | 1.50 MiB/s, done.

-----> Building on the Heroku-20 stack
-----> Using buildpack: heroku/nodejs
-----> Node.js app detected

-----> Installing node modules
       Installing dependencies from package-lock.json
       added 1234 packages in 45s

-----> Building Nuxt.js application
       > nuxt build
       ✓ Nuxt build complete

-----> Discovering process types
       Procfile declares types -> web

-----> Compressing...
       Done: 45.6M

-----> Launching...
       Released v1
       https://my-nuxt-app.herokuapp.com/ deployed to Heroku
```

### 後續部署

```bash
# 1. 修改代碼
# 2. 提交變更
git add .
git commit -m "Update feature"

# 3. 推送到 Heroku
git push heroku main

# Heroku 會自動：
# ✓ 檢測變更
# ✓ 安裝依賴
# ✓ 構建應用
# ✓ 重啟 dynos
```

### 從非 main 分支部署

```bash
# 從 develop 分支部署
git push heroku develop:main

# 從本地分支部署
git push heroku feature-branch:main
```

### 回滾部署

```bash
# 查看部署歷史
heroku releases

# 回滾到上一個版本
heroku rollback

# 回滾到特定版本
heroku rollback v123
```

---

## 環境變數設定（Config Vars）

### 使用 Heroku CLI 設定

```bash
# 設定單個環境變數
heroku config:set NODE_ENV=production

# 設定多個環境變數
heroku config:set \
  NUXT_PUBLIC_API_BASE=https://api.example.com \
  API_SECRET=your-secret-key \
  DATABASE_URL=postgres://...

# 查看所有環境變數
heroku config

# 查看特定環境變數
heroku config:get API_SECRET

# 刪除環境變數
heroku config:unset API_SECRET

# 從 .env 文件批量導入
heroku config:set $(cat .env | sed 's/#.*//g' | xargs)
```

### 使用 Heroku Dashboard 設定

1. 訪問 https://dashboard.heroku.com/
2. 選擇你的應用
3. 點擊 **Settings** 標籤
4. 找到 **Config Vars** 區塊
5. 點擊 **Reveal Config Vars**
6. 添加環境變數：
   ```
   KEY                    VALUE
   ─────────────────────────────────────────
   NODE_ENV               production
   NUXT_PUBLIC_API_BASE   https://api.example.com
   API_SECRET             your-secret-key
   DATABASE_URL           postgres://...
   ```

### 在應用中使用環境變數

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    // 私有變數
    apiSecret: process.env.API_SECRET,
    databaseUrl: process.env.DATABASE_URL,

    public: {
      // 公開變數
      apiBase: process.env.NUXT_PUBLIC_API_BASE
    }
  }
})

// 在代碼中使用
// composables/useApi.ts
export const useApi = () => {
  const config = useRuntimeConfig()

  return {
    apiBase: config.public.apiBase,
    apiSecret: config.apiSecret // 僅服務器端可訪問
  }
}
```

### Heroku 系統環境變數

Heroku 自動提供以下環境變數：

```typescript
// 端口（必須使用）
process.env.PORT // 動態分配的端口

// 其他系統變數
process.env.DYNO // dyno 名稱，如 "web.1"
process.env.HEROKU_APP_NAME // 應用名稱
process.env.HEROKU_RELEASE_VERSION // 發布版本，如 "v123"
process.env.HEROKU_SLUG_COMMIT // Git commit SHA
```

---

## 資料庫整合

### PostgreSQL (推薦)

#### 1. 添加 Heroku Postgres

```bash
# 添加 Postgres add-on（Eco 方案，$5/月）
heroku addons:create heroku-postgresql:essential-0

# 或 Mini 方案（$5/月，20GB）
heroku addons:create heroku-postgresql:mini

# 查看資料庫資訊
heroku pg:info

# 查看 DATABASE_URL
heroku config:get DATABASE_URL
```

#### 2. 安裝 Postgres 客戶端

```bash
npm install pg
# 或
pnpm add pg
```

#### 3. 連接資料庫

```typescript
// server/utils/db.ts
import pg from 'pg'

const { Pool } = pg

let pool: pg.Pool | null = null

export const getDb = () => {
  if (!pool) {
    const config = useRuntimeConfig()
    pool = new Pool({
      connectionString: config.databaseUrl,
      ssl: {
        rejectUnauthorized: false
      }
    })
  }
  return pool
}

// server/api/users.ts
export default defineEventHandler(async (event) => {
  const db = getDb()

  const result = await db.query('SELECT * FROM users LIMIT 10')

  return result.rows
})
```

#### 4. 運行資料庫遷移

```bash
# 連接到資料庫
heroku pg:psql

# 執行 SQL
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(100) UNIQUE,
  created_at TIMESTAMP DEFAULT NOW()
);

# 或使用遷移工具（Prisma）
npx prisma migrate deploy
```

### Redis

```bash
# 添加 Redis add-on
heroku addons:create heroku-redis:mini

# 安裝 Redis 客戶端
npm install redis

# 使用 Redis
```

```typescript
// server/utils/redis.ts
import { createClient } from 'redis'

let client: ReturnType<typeof createClient> | null = null

export const getRedis = async () => {
  if (!client) {
    const config = useRuntimeConfig()
    client = createClient({
      url: config.redisUrl
    })
    await client.connect()
  }
  return client
}

// server/api/cache.ts
export default defineEventHandler(async (event) => {
  const redis = await getRedis()

  // 設定快取
  await redis.set('key', 'value', { EX: 3600 })

  // 讀取快取
  const value = await redis.get('key')

  return { value }
})
```

### MongoDB

```bash
# 不推薦使用 Heroku 的 MongoDB add-on（已停用）
# 建議使用 MongoDB Atlas（免費）

# 在 MongoDB Atlas 建立叢集後，設定環境變數
heroku config:set MONGODB_URI=mongodb+srv://...
```

---

## Add-ons 使用

Heroku 提供豐富的 Add-ons 生態系統。

### 常用 Add-ons

#### 1. Papertrail（日誌管理）

```bash
# 添加 Papertrail（免費方案）
heroku addons:create papertrail:choklad

# 查看日誌
heroku addons:open papertrail
```

#### 2. SendGrid（郵件服務）

```bash
# 添加 SendGrid（免費方案：每月 12,000 封）
heroku addons:create sendgrid:starter

# 查看 API Key
heroku config:get SENDGRID_API_KEY
```

```typescript
// server/api/send-email.ts
import sgMail from '@sendgrid/mail'

export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  sgMail.setApiKey(config.sendgridApiKey)

  const msg = {
    to: 'user@example.com',
    from: 'noreply@example.com',
    subject: 'Hello from Nuxt!',
    text: 'This is a test email',
    html: '<strong>This is a test email</strong>'
  }

  await sgMail.send(msg)

  return { success: true }
})
```

#### 3. Cloudinary（圖片管理）

```bash
# 添加 Cloudinary（免費方案）
heroku addons:create cloudinary:starter

# 查看憑證
heroku config | grep CLOUDINARY
```

#### 4. New Relic（APM 監控）

```bash
# 添加 New Relic（免費方案）
heroku addons:create newrelic:wayne

# 查看監控
heroku addons:open newrelic
```

### 管理 Add-ons

```bash
# 列出所有 add-ons
heroku addons

# 查看特定 add-on 資訊
heroku addons:info heroku-postgresql

# 升級 add-on
heroku addons:upgrade heroku-postgresql:standard-0

# 刪除 add-on
heroku addons:destroy heroku-postgresql
```

---

## 自定義域名

### 添加自定義域名

```bash
# 添加域名
heroku domains:add www.example.com

# 添加 apex 域名（需要付費方案）
heroku domains:add example.com

# 查看所有域名
heroku domains

# 刪除域名
heroku domains:remove www.example.com
```

### 配置 DNS

添加域名後，Heroku 會提供 DNS 目標：

```
Domain Name      DNS Target
───────────────────────────────────────────────────
www.example.com  flowing-mountain-1234.herokudns.com
```

在你的 DNS 提供商配置：

```
Type: CNAME
Name: www
Value: flowing-mountain-1234.herokudns.com
```

對於 apex 域名（example.com）：

```
Type: ALIAS 或 ANAME
Name: @
Value: flowing-mountain-1234.herokudns.com

# 或使用 A 記錄（不推薦，IP 可能變更）
```

### SSL 證書

Heroku 自動提供免費的 SSL 證書（Let's Encrypt）：

```bash
# 查看 SSL 狀態
heroku certs:auto

# 啟用自動 SSL（預設已啟用）
heroku certs:auto:enable

# 查看證書資訊
heroku certs
```

### 強制 HTTPS

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/**': {
      headers: {
        'Strict-Transport-Security': 'max-age=31536000; includeSubDomains'
      }
    }
  }
})
```

或使用中間件：

```typescript
// server/middleware/force-https.ts
export default defineEventHandler((event) => {
  const host = getRequestHeader(event, 'host')
  const proto = getRequestHeader(event, 'x-forwarded-proto')

  if (proto === 'http' && host?.includes('example.com')) {
    return sendRedirect(event, `https://${host}${event.node.req.url}`, 301)
  }
})
```

---

## 日誌查看

### 即時日誌

```bash
# 查看即時日誌
heroku logs --tail

# 只查看 app 日誌
heroku logs --source app --tail

# 只查看最近 200 行
heroku logs -n 200

# 過濾日誌
heroku logs --tail | grep ERROR
```

### 日誌輸出範例

```
2024-11-16T10:30:45.123456+00:00 app[web.1]: Server listening on http://0.0.0.0:12345
2024-11-16T10:30:46.234567+00:00 heroku[web.1]: State changed from starting to up
2024-11-16T10:31:00.345678+00:00 app[web.1]: GET / 200 45ms
2024-11-16T10:31:05.456789+00:00 app[web.1]: GET /api/users 200 123ms
```

### 使用 Papertrail

```bash
# 添加 Papertrail
heroku addons:create papertrail

# 打開 Papertrail Dashboard
heroku addons:open papertrail
```

Papertrail 提供：
- 即時日誌流
- 搜索和過濾
- 警報和通知
- 日誌保留（7 天到永久）

---

## Dyno 管理

### Dyno 類型

| 類型 | RAM | 價格 | 適用場景 |
|------|-----|------|---------|
| Eco | 512MB | $5/月（共享） | 個人項目、開發環境 |
| Basic | 512MB | $7/月 | 簡單應用 |
| Standard-1X | 512MB | $25/月 | 生產應用 |
| Standard-2X | 1GB | $50/月 | 中型應用 |
| Performance-M | 2.5GB | $250/月 | 高流量應用 |
| Performance-L | 14GB | $500/月 | 大型應用 |

### Dyno 命令

```bash
# 查看 dyno 狀態
heroku ps

# 擴展 dyno（增加數量）
heroku ps:scale web=2

# 調整 dyno 類型
heroku ps:type standard-1x

# 重啟 dyno
heroku ps:restart

# 重啟特定 dyno
heroku ps:restart web.1

# 停止 dyno
heroku ps:stop web.1
```

### 自動擴展

只有 Performance dyno 支援自動擴展：

```bash
# 啟用自動擴展
heroku ps:autoscale:enable --min 2 --max 10

# 禁用自動擴展
heroku ps:autoscale:disable
```

---

## 完整部署步驟

### 準備工作

```bash
# 1. 確保項目已初始化 Git
git init

# 2. 建立 Procfile
echo "web: node .output/server/index.mjs" > Procfile

# 3. 確保 package.json 正確配置
# 檢查 "engines" 和 "scripts"

# 4. 配置 nuxt.config.ts
# 確保 preset 設為 'node-server'

# 5. 本地測試
npm run build
npm run start
```

### 建立和部署

```bash
# 6. 安裝 Heroku CLI（如果還沒有）
brew install heroku/brew/heroku

# 7. 登入 Heroku
heroku login

# 8. 建立 Heroku 應用
heroku create my-nuxt-app

# 9. 提交代碼
git add .
git commit -m "Initial commit"

# 10. 推送到 Heroku
git push heroku main

# 11. 等待部署完成
# Heroku 會自動：
# - 檢測 Node.js 項目
# - 安裝依賴
# - 運行構建
# - 啟動應用

# 12. 打開應用
heroku open
```

### 配置環境變數

```bash
# 13. 設定環境變數
heroku config:set \
  NODE_ENV=production \
  NUXT_PUBLIC_API_BASE=https://api.example.com

# 14. 如果使用資料庫
heroku addons:create heroku-postgresql:mini

# 15. 查看應用狀態
heroku ps
heroku logs --tail
```

### 設定自定義域名

```bash
# 16. 添加域名
heroku domains:add www.example.com

# 17. 配置 DNS（根據 Heroku 提供的目標）

# 18. 等待 SSL 證書生成
heroku certs:auto
```

---

## 疑難排解

### 問題 1：應用無法啟動

**錯誤：** Application error

**常見原因：**

```bash
# 1. 查看日誌
heroku logs --tail

# 2. 檢查 Procfile
# 確保命令正確
web: node .output/server/index.mjs

# 3. 檢查端口
# Nuxt 會自動使用 process.env.PORT
# 無需手動配置

# 4. 檢查 package.json
# 確保有 "start" script
{
  "scripts": {
    "start": "node .output/server/index.mjs"
  }
}
```

### 問題 2：構建失敗

**錯誤：** Build failed

**解決方案：**

```bash
# 1. 檢查 Node.js 版本
# package.json
{
  "engines": {
    "node": "18.x"
  }
}

# 2. 清除構建快取
heroku plugins:install heroku-builds
heroku builds:cache:purge

# 3. 檢查依賴
npm install
npm run build

# 4. 確保所有依賴在 dependencies 而非 devDependencies
# 如果構建需要，移到 dependencies
```

### 問題 3：記憶體溢出

**錯誤：** R14 - Memory quota exceeded

**解決方案：**

```bash
# 1. 升級 dyno 類型
heroku ps:type standard-2x

# 2. 優化構建
# package.json
{
  "scripts": {
    "build": "NODE_OPTIONS='--max-old-space-size=4096' nuxt build"
  }
}

# 3. 減少記憶體使用
# 啟用壓縮、最小化等
```

### 問題 4：資料庫連接失敗

**解決方案：**

```typescript
// 確保使用 SSL
import pg from 'pg'

const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: {
    rejectUnauthorized: false // Heroku Postgres 需要
  }
})
```

### 問題 5：靜態資源 404

**問題：** CSS、JS 文件無法載入

**解決方案：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  app: {
    baseURL: '/',
    buildAssetsDir: '_nuxt/'
  }
})
```

### 問題 6：環境變數未生效

**解決方案：**

```bash
# 1. 確認環境變數已設定
heroku config

# 2. 檢查代碼
# ❌ 錯誤
const apiKey = process.env.API_KEY

# ✅ 正確
const config = useRuntimeConfig()
const apiKey = config.apiKey

# 3. 重啟 dyno
heroku ps:restart
```

### 除錯技巧

```bash
# 1. 查看詳細日誌
heroku logs --tail --source app

# 2. 運行 bash
heroku run bash
ls -la
cat .output/server/index.mjs

# 3. 檢查環境
heroku run env

# 4. 測試構建
heroku run npm run build

# 5. 查看 dyno 資訊
heroku ps
heroku ps:exec
```

---

## 效能優化

### 1. 啟用 HTTP/2

```bash
# Heroku 自動支援 HTTP/2
# 確保使用 HTTPS
```

### 2. 使用 CDN

```bash
# 添加 Cloudflare（免費）
heroku addons:create cloudflare

# 或使用 Fastly
heroku addons:create fastly
```

### 3. 快取策略

```typescript
// server/api/cached.ts
export default defineEventHandler((event) => {
  setResponseHeaders(event, {
    'Cache-Control': 'public, max-age=3600, s-maxage=86400',
    'Vary': 'Accept-Encoding'
  })

  return {
    data: 'This response is cached'
  }
})
```

### 4. 壓縮

```bash
# 安裝壓縮中間件
npm install compression
```

```typescript
// server/middleware/compression.ts
import compression from 'compression'

export default fromNodeMiddleware(compression())
```

### 5. 預載入關鍵資源

```vue
<!-- app.vue -->
<script setup>
useHead({
  link: [
    { rel: 'preconnect', href: 'https://api.example.com' },
    { rel: 'dns-prefetch', href: 'https://api.example.com' }
  ]
})
</script>
```

---

## CI/CD 整合

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Heroku

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Deploy to Heroku
        uses: akhileshns/heroku-deploy@v3.13.15
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: 'my-nuxt-app'
          heroku_email: 'your-email@example.com'
```

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - test
  - deploy

test:
  stage: test
  image: node:18
  script:
    - npm ci
    - npm run lint
    - npm run test

deploy:
  stage: deploy
  image: ruby:latest
  only:
    - main
  script:
    - gem install dpl
    - dpl --provider=heroku --app=my-nuxt-app --api-key=$HEROKU_API_KEY
```

---

## 總結

### Heroku 優點

✅ **簡單易用**：Git push 即可部署
✅ **豐富的 Add-ons**：一鍵添加資料庫、快取等
✅ **完善的文檔**：豐富的學習資源
✅ **自動擴展**：根據負載調整
✅ **企業級支援**：適合商業應用
✅ **多語言支援**：不限於 Node.js

### Heroku 缺點

❌ **無免費方案**：最低 $5/月
❌ **價格較高**：相比其他平台
❌ **Dyno 休眠**：Eco dyno 會休眠
❌ **有限的自定義**：無法完全控制基礎設施
❌ **地區限制**：只有美國和歐洲

### 費用比較

```
小型項目（1 dyno + Postgres）:
- Eco dyno: $5/月
- Essential Postgres: $5/月
總計: $10/月

中型項目（2 dynos + Postgres + Redis）:
- Standard-1X dyno × 2: $50/月
- Standard Postgres: $50/月
- Mini Redis: $15/月
總計: $115/月

對比：
- Vercel: $0-20/月
- Cloudflare Pages: $0/月
- GCP Cloud Run: $5-20/月
```

### 推薦使用場景

1. **需要資料庫的全棧應用** - Add-ons 整合方便
2. **原型和 MVP** - 快速部署和迭代
3. **企業應用**（有預算）- 企業級支援
4. **熟悉 Heroku 的團隊** - 降低學習成本

### 替代方案

如果 Heroku 太貴，考慮：
- **Render**：類似 Heroku，更便宜（$7/月包含資料庫）
- **Railway**：現代化 PaaS（$5/月起）
- **Fly.io**：全球部署（$3/月起）
- **DigitalOcean App Platform**：$5/月起

### 最佳實踐

1. 使用環境變數管理配置
2. 啟用自動 SSL
3. 定期備份資料庫
4. 使用 Add-ons 簡化整合
5. 監控日誌和效能
6. 設定適當的 dyno 類型
7. 實施 CI/CD 自動化

---

## 參考資源

- [Heroku 文檔](https://devcenter.heroku.com/)
- [Heroku Node.js 支援](https://devcenter.heroku.com/articles/nodejs-support)
- [Heroku Postgres](https://devcenter.heroku.com/articles/heroku-postgresql)
- [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)
- [Heroku Add-ons](https://elements.heroku.com/addons)
- [Heroku Pricing](https://www.heroku.com/pricing)
- [Nuxt Deployment](https://nuxt.com/docs/getting-started/deployment)
