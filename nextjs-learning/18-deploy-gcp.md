# 第 18 章：部署到 Google Cloud Platform (Cloud Run / App Engine)

## 目錄
- [簡介](#簡介)
- [GCP 服務選擇](#gcp-服務選擇)
- [定價與免費額度](#定價與免費額度)
- [前置準備](#前置準備)
- [部署到 Cloud Run](#部署到-cloud-run)
- [部署到 App Engine](#部署到-app-engine)
- [環境變數設定](#環境變數設定)
- [自訂網域與 SSL](#自訂網域與-ssl)
- [效能優化](#效能優化)
- [監控與日誌](#監控與日誌)
- [故障排除](#故障排除)

---

## 簡介

Google Cloud Platform (GCP) 提供多種服務來部署 Next.js 應用程式。本章將介紹兩種最常用的服務：

1. **Cloud Run**：容器化的無伺服器平台（推薦）
2. **App Engine**：全託管的應用程式平台

## GCP 服務選擇

### Cloud Run vs App Engine

| 特性 | Cloud Run | App Engine |
|------|-----------|------------|
| **架構** | 容器化（Docker） | 平台即服務（PaaS） |
| **彈性** | 高度客製化 | 標準化環境 |
| **冷啟動** | 較快 | 較慢 |
| **定價** | 按請求計費 | 按實例時數計費 |
| **擴展** | 自動（0 到 N） | 自動（最小 1 個實例） |
| **適用場景** | 容器化應用、微服務 | 傳統 Web 應用 |
| **複雜度** | 中等 | 簡單 |

### 推薦選擇

- **Cloud Run**：適合現代化應用、需要精細控制、成本敏感
- **App Engine**：適合快速部署、不想管理容器、傳統架構

## 定價與免費額度

### Cloud Run 定價

#### 免費額度（每月）
- **請求數**：200 萬次
- **CPU 時間**：180,000 vCPU 秒
- **記憶體**：360,000 GiB 秒
- **網路流出**：1 GB（北美）

#### 付費價格（超過免費額度後）
- **CPU**：$0.00002400/vCPU 秒
- **記憶體**：$0.00000250/GiB 秒
- **請求數**：$0.40/百萬次
- **網路流出**：$0.12/GB

#### 成本估算範例
小型應用（10,000 次請求/月）：
- CPU: 256m vCPU, 512MB RAM
- 平均回應時間: 200ms
- **預估成本**：$0（在免費額度內）

中型應用（100 萬次請求/月）：
- **預估成本**：$5-10/月

### App Engine 定價

#### 免費額度（每日）
- **實例時數**：28 小時（F1/F2 實例）
- **流出流量**：1 GB
- **流入流量**：免費無限制

#### 標準環境定價
- **F1 實例**（256MB RAM）：$0.05/小時
- **F2 實例**（512MB RAM）：$0.10/小時
- **F4 實例**（1GB RAM）：$0.20/小時

#### 成本估算範例
單一 F1 實例全天運行：
- 24 小時 × $0.05 = **$1.20/天**
- 每月約 **$36**

## 前置準備

### 1. 建立 GCP 帳號

1. 前往 [Google Cloud Console](https://console.cloud.google.com/)
2. 登入或建立 Google 帳號
3. 啟用免費試用（$300 信用額度，90 天有效）
4. 輸入付款資訊（不會自動收費）

### 2. 建立專案

```bash
# 安裝 Google Cloud SDK
# macOS
brew install --cask google-cloud-sdk

# Windows
# 下載安裝程式：https://cloud.google.com/sdk/docs/install

# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# 初始化 gcloud
gcloud init

# 建立新專案
gcloud projects create my-nextjs-app --name="My Next.js App"

# 設定為目前專案
gcloud config set project my-nextjs-app

# 啟用必要的 API
gcloud services enable cloudbuild.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable containerregistry.googleapis.com
```

### 3. 安裝必要工具

```bash
# 驗證安裝
gcloud --version
docker --version

# 設定 Docker 認證
gcloud auth configure-docker
```

### 4. 準備 Next.js 專案

確保專案可以在 Node.js 環境中運行：

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
  }
}
```

## 部署到 Cloud Run

### 方法一：使用 Dockerfile（推薦）

#### 步驟 1：建立 Dockerfile

在專案根目錄建立 `Dockerfile`：

```dockerfile
# 多階段建置以減少映像大小

# 階段 1: 依賴項安裝
FROM node:18-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# 只複製 package 檔案
COPY package.json package-lock.json* ./
RUN npm ci

# 階段 2: 建置應用程式
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# 設定環境變數
ENV NEXT_TELEMETRY_DISABLED 1

# 建置 Next.js
RUN npm run build

# 階段 3: 生產執行環境
FROM node:18-alpine AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# 複製必要檔案
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

USER nextjs

EXPOSE 8080

ENV PORT 8080
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

#### 步驟 2：設定 Next.js 輸出模式

```javascript
// next.config.js
module.exports = {
  output: 'standalone',

  // 可選：自訂伺服器配置
  compress: true,

  // 環境變數
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
  },
}
```

#### 步驟 3：建立 .dockerignore

```
# .dockerignore
node_modules
.next
.git
.env*.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.DS_Store
README.md
```

#### 步驟 4：建置並推送 Docker 映像

```bash
# 設定變數
PROJECT_ID=$(gcloud config get-value project)
SERVICE_NAME="my-nextjs-app"
REGION="asia-east1"  # 台灣

# 建置映像
gcloud builds submit \
  --tag gcr.io/$PROJECT_ID/$SERVICE_NAME

# 或使用 Docker 手動建置
docker build -t gcr.io/$PROJECT_ID/$SERVICE_NAME .
docker push gcr.io/$PROJECT_ID/$SERVICE_NAME
```

#### 步驟 5：部署到 Cloud Run

```bash
# 部署服務
gcloud run deploy $SERVICE_NAME \
  --image gcr.io/$PROJECT_ID/$SERVICE_NAME \
  --platform managed \
  --region $REGION \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \
  --max-instances 10 \
  --port 8080

# 部署成功後會顯示服務 URL
# Service URL: https://my-nextjs-app-xxxxx-uc.a.run.app
```

#### 步驟 6：查看部署狀態

```bash
# 列出所有服務
gcloud run services list

# 查看服務詳情
gcloud run services describe $SERVICE_NAME --region $REGION

# 查看日誌
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=$SERVICE_NAME" \
  --limit 50 \
  --format json
```

### 方法二：使用 Cloud Build 自動化

#### 步驟 1：建立 cloudbuild.yaml

```yaml
# cloudbuild.yaml
steps:
  # 建置 Docker 映像
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'gcr.io/$PROJECT_ID/my-nextjs-app:$COMMIT_SHA'
      - '-t'
      - 'gcr.io/$PROJECT_ID/my-nextjs-app:latest'
      - '.'

  # 推送映像到 Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'gcr.io/$PROJECT_ID/my-nextjs-app:$COMMIT_SHA'

  # 部署到 Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'my-nextjs-app'
      - '--image'
      - 'gcr.io/$PROJECT_ID/my-nextjs-app:$COMMIT_SHA'
      - '--region'
      - 'asia-east1'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'
      - '--memory'
      - '512Mi'

images:
  - 'gcr.io/$PROJECT_ID/my-nextjs-app:$COMMIT_SHA'
  - 'gcr.io/$PROJECT_ID/my-nextjs-app:latest'

options:
  logging: CLOUD_LOGGING_ONLY
```

#### 步驟 2：執行建置

```bash
# 手動觸發建置
gcloud builds submit --config cloudbuild.yaml

# 查看建置歷史
gcloud builds list --limit=10

# 查看特定建置的日誌
gcloud builds log [BUILD_ID]
```

#### 步驟 3：設定 GitHub 自動部署

```bash
# 連結 GitHub 儲存庫
gcloud builds triggers create github \
  --repo-name=my-nextjs-app \
  --repo-owner=your-username \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml

# 查看觸發器
gcloud builds triggers list
```

### 方法三：使用 Google Cloud Console 介面

1. 前往 [Cloud Run Console](https://console.cloud.google.com/run)
2. 點擊「建立服務」
3. 選擇「從 GitHub 部署一個修訂版本」
4. 連結 GitHub 帳號並選擇儲存庫
5. 設定建置配置：
   - **Branch**: main
   - **Build Type**: Dockerfile
   - **Dockerfile path**: Dockerfile
6. 設定服務：
   - **Service name**: my-nextjs-app
   - **Region**: asia-east1
   - **CPU allocation**: 僅在處理請求期間配置
   - **Memory**: 512 MiB
   - **Maximum instances**: 10
7. 點擊「建立」

## 部署到 App Engine

### 步驟 1：建立 app.yaml

在專案根目錄建立 `app.yaml`：

```yaml
# app.yaml
runtime: nodejs18

# 實例類型
instance_class: F1

# 自動擴展設定
automatic_scaling:
  min_instances: 1
  max_instances: 5
  min_idle_instances: 1
  max_idle_instances: automatic
  min_pending_latency: 30ms
  max_pending_latency: automatic
  max_concurrent_requests: 80

# 環境變數
env_variables:
  NODE_ENV: 'production'
  HOST: '0.0.0.0'

# 處理靜態檔案
handlers:
  # 靜態資源
  - url: /_next/static
    static_dir: .next/static
    secure: always
    expiration: "1y"

  # 公開檔案
  - url: /(.+\.(ico|png|jpg|jpeg|gif|svg|webp))$
    static_files: public/\1
    upload: public/.*\.(ico|png|jpg|jpeg|gif|svg|webp)$
    secure: always

  # 所有其他請求
  - url: /.*
    script: auto
    secure: always
```

### 步驟 2：建立 .gcloudignore

```
# .gcloudignore
.gcloudignore
.git
.gitignore
node_modules/
.next/cache/
.env*.local
```

### 步驟 3：修改 package.json

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start -p $PORT",
    "gcp-build": "next build"
  },
  "engines": {
    "node": "18.x"
  }
}
```

### 步驟 4：部署

```bash
# 初始化 App Engine（首次部署）
gcloud app create --region=asia-east1

# 部署應用程式
gcloud app deploy

# 系統會詢問：
# Do you want to continue (Y/n)? Y

# 部署完成後開啟應用程式
gcloud app browse
```

### 步驟 5：管理版本

```bash
# 列出所有版本
gcloud app versions list

# 設定流量分配
gcloud app services set-traffic default \
  --splits v1=0.5,v2=0.5

# 刪除舊版本
gcloud app versions delete v1

# 查看日誌
gcloud app logs tail -s default
```

### App Engine 進階設定

#### 使用彈性環境

```yaml
# app.yaml (彈性環境)
runtime: nodejs
env: flex

# 資源配置
resources:
  cpu: 1
  memory_gb: 0.5
  disk_size_gb: 10

# 網路設定
network:
  session_affinity: true

# 自動擴展
automatic_scaling:
  min_num_instances: 1
  max_num_instances: 5
  cool_down_period_sec: 120
  cpu_utilization:
    target_utilization: 0.6

# 健康檢查
liveness_check:
  path: "/api/health"
  check_interval_sec: 30
  timeout_sec: 4
  failure_threshold: 2
  success_threshold: 2

readiness_check:
  path: "/api/ready"
  check_interval_sec: 5
  timeout_sec: 4
  failure_threshold: 2
  success_threshold: 2
  app_start_timeout_sec: 300
```

#### 建立健康檢查端點

```javascript
// pages/api/health.js
export default function handler(req, res) {
  res.status(200).json({ status: 'healthy' })
}

// pages/api/ready.js
export default function handler(req, res) {
  // 檢查資料庫連線等
  res.status(200).json({ status: 'ready' })
}
```

## 環境變數設定

### Cloud Run 環境變數

#### 透過指令設定

```bash
# 設定環境變數
gcloud run services update my-nextjs-app \
  --region asia-east1 \
  --set-env-vars "DATABASE_URL=postgresql://...,API_KEY=xxx"

# 從檔案讀取
gcloud run services update my-nextjs-app \
  --region asia-east1 \
  --env-vars-file .env.yaml
```

`.env.yaml` 格式：

```yaml
DATABASE_URL: "postgresql://..."
API_KEY: "your-api-key"
NEXT_PUBLIC_API_URL: "https://api.example.com"
```

#### 使用 Secret Manager

```bash
# 建立 Secret
echo -n "my-secret-value" | gcloud secrets create my-secret --data-file=-

# 授予 Cloud Run 存取權限
gcloud secrets add-iam-policy-binding my-secret \
  --member=serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor

# 部署時使用 Secret
gcloud run deploy my-nextjs-app \
  --image gcr.io/$PROJECT_ID/my-nextjs-app \
  --region asia-east1 \
  --set-secrets "DATABASE_URL=my-secret:latest"
```

在應用程式中使用：

```javascript
// 直接讀取（已自動注入為環境變數）
const dbUrl = process.env.DATABASE_URL
```

### App Engine 環境變數

#### 在 app.yaml 中設定

```yaml
env_variables:
  DATABASE_URL: "postgresql://..."
  API_KEY: "your-api-key"
  NEXT_PUBLIC_API_URL: "https://api.example.com"
```

#### 使用 Secret Manager

```yaml
# app.yaml
env_variables:
  # 一般環境變數
  NODE_ENV: "production"

# 從 Secret Manager 讀取
secret_env_variables:
  - key: DATABASE_URL
    secret: database-url
    version: latest
```

建立 Secret：

```bash
echo -n "postgresql://..." | gcloud secrets create database-url --data-file=-

# 授予 App Engine 存取權限
gcloud secrets add-iam-policy-binding database-url \
  --member=serviceAccount:PROJECT_ID@appspot.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

## 自訂網域與 SSL

### Cloud Run 自訂網域

#### 步驟 1：驗證網域所有權

```bash
# 新增網域對應
gcloud beta run domain-mappings create \
  --service my-nextjs-app \
  --domain www.example.com \
  --region asia-east1
```

#### 步驟 2：設定 DNS

在 DNS 提供商新增以下記錄：

```
# CNAME 記錄
Type: CNAME
Name: www
Value: ghs.googlehosted.com
```

或使用 A 記錄：

```
Type: A
Name: @
Value: [Cloud Run 提供的 IP 位址]
```

#### 步驟 3：啟用 HTTPS

Cloud Run 會自動配置 SSL 憑證（Let's Encrypt），無需手動設定。

#### 步驟 4：查看對應狀態

```bash
# 列出所有網域對應
gcloud beta run domain-mappings list --region asia-east1

# 查看特定對應
gcloud beta run domain-mappings describe www.example.com --region asia-east1
```

### App Engine 自訂網域

#### 步驟 1：新增自訂網域

```bash
# 使用 Cloud Console 或指令
gcloud app domain-mappings create www.example.com

# 系統會提供驗證記錄
```

#### 步驟 2：驗證網域

在 DNS 新增驗證記錄（TXT 記錄）：

```
Type: TXT
Name: @
Value: google-site-verification=xxxxx
```

#### 步驟 3：設定 DNS

```
Type: A
Name: @
Value: 216.239.32.21
Value: 216.239.34.21
Value: 216.239.36.21
Value: 216.239.38.21

Type: AAAA (IPv6，選填)
Name: @
Value: 2001:4860:4802:32::15
Value: 2001:4860:4802:34::15
Value: 2001:4860:4802:36::15
Value: 2001:4860:4802:38::15
```

對於子網域：

```
Type: CNAME
Name: www
Value: ghs.googlehosted.com
```

#### 步驟 4：啟用 SSL

App Engine 自動提供 Google 管理的 SSL 憑證，會自動續期。

## 效能優化

### 1. Cloud Run 優化

#### 減少冷啟動時間

```bash
# 設定最小實例數（需付費）
gcloud run services update my-nextjs-app \
  --region asia-east1 \
  --min-instances 1
```

#### 優化 Docker 映像

```dockerfile
# 使用多階段建置
FROM node:18-alpine AS base

# 安裝依賴
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# 建置
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# 執行
FROM base AS runner
WORKDIR /app
ENV NODE_ENV production

# 只複製必要檔案
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 8080
CMD ["node", "server.js"]
```

#### 調整資源配置

```bash
gcloud run services update my-nextjs-app \
  --region asia-east1 \
  --memory 1Gi \
  --cpu 2 \
  --concurrency 80 \
  --timeout 300
```

### 2. App Engine 優化

#### 調整實例類型

```yaml
# app.yaml
instance_class: F2  # 更大的實例

automatic_scaling:
  max_concurrent_requests: 80
  target_cpu_utilization: 0.6
  target_throughput_utilization: 0.6
```

#### 快取靜態資源

```yaml
handlers:
  - url: /_next/static
    static_dir: .next/static
    secure: always
    expiration: "365d"
    http_headers:
      Cache-Control: "public, max-age=31536000, immutable"
```

### 3. Next.js 優化

```javascript
// next.config.js
module.exports = {
  // 壓縮
  compress: true,

  // 移除 console（生產環境）
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production',
  },

  // 圖片優化
  images: {
    formats: ['image/avif', 'image/webp'],
    minimumCacheTTL: 60,
  },

  // SWC 最小化
  swcMinify: true,

  // 輸出設定
  output: 'standalone',
}
```

### 4. 使用 Cloud CDN

```bash
# 啟用 Cloud CDN（需要負載平衡器）
gcloud compute backend-services update my-backend \
  --enable-cdn \
  --cache-mode CACHE_ALL_STATIC
```

## 監控與日誌

### Cloud Run 監控

#### 查看指標

```bash
# 在 Cloud Console 查看
# https://console.cloud.google.com/run/detail/[REGION]/[SERVICE]/metrics

# 使用 gcloud 查看日誌
gcloud logging read "resource.type=cloud_run_revision" \
  --limit 50 \
  --format json
```

#### 設定告警

```bash
# 建立告警政策（使用 Cloud Console）
# Monitoring > Alerting > Create Policy

# 常見告警：
# - 請求延遲 > 1000ms
# - 錯誤率 > 5%
# - CPU 使用率 > 80%
# - 記憶體使用率 > 80%
```

#### 啟用 Cloud Trace

```javascript
// 安裝套件
npm install @google-cloud/trace-agent

// 在應用程式啟動時啟用
// server.js 或 pages/_app.js
require('@google-cloud/trace-agent').start()
```

### App Engine 監控

#### 查看日誌

```bash
# 即時日誌
gcloud app logs tail -s default

# 查看特定時間範圍
gcloud app logs read --service default --limit 100

# 使用 Logs Explorer
# https://console.cloud.google.com/logs
```

#### 應用程式診斷

```bash
# 啟用 Cloud Debugger
npm install @google-cloud/debug-agent

# 在應用程式中啟用
require('@google-cloud/debug-agent').start()
```

### 使用 Cloud Monitoring

#### 自訂指標

```javascript
// 安裝套件
npm install @google-cloud/monitoring

// 記錄自訂指標
const monitoring = require('@google-cloud/monitoring')
const client = new monitoring.MetricServiceClient()

async function writeCustomMetric(value) {
  const projectId = await client.getProjectId()
  const dataPoint = {
    interval: {
      endTime: {
        seconds: Date.now() / 1000,
      },
    },
    value: {
      doubleValue: value,
    },
  }

  const timeSeriesData = {
    metric: {
      type: 'custom.googleapis.com/my_metric',
    },
    resource: {
      type: 'global',
    },
    points: [dataPoint],
  }

  const request = {
    name: client.projectPath(projectId),
    timeSeries: [timeSeriesData],
  }

  await client.createTimeSeries(request)
}
```

## 故障排除

### 常見問題

#### 1. Cloud Run: 容器啟動失敗

**錯誤**：`Container failed to start. Failed to start and then listen on the port defined by the PORT environment variable.`

**解決方法**：

確保應用程式監聽 `PORT` 環境變數：

```javascript
// server.js
const PORT = process.env.PORT || 8080
app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`)
})
```

Dockerfile 設定：

```dockerfile
ENV PORT 8080
EXPOSE 8080
```

#### 2. Cloud Run: 記憶體不足

**錯誤**：`Memory limit exceeded`

**解決方法**：

```bash
# 增加記憶體配置
gcloud run services update my-nextjs-app \
  --region asia-east1 \
  --memory 2Gi
```

優化 Node.js 記憶體：

```dockerfile
ENV NODE_OPTIONS="--max-old-space-size=2048"
```

#### 3. App Engine: 建置失敗

**錯誤**：`ERROR: (gcloud.app.deploy) Error Response: [9] Cloud build failed.`

**解決方法**：

查看詳細錯誤：

```bash
# 查看 Cloud Build 日誌
gcloud builds list --limit 1
gcloud builds log [BUILD_ID]
```

常見原因：
- `package.json` 中缺少 `gcp-build` 腳本
- Node.js 版本不相容
- 依賴項安裝失敗

修正：

```json
{
  "scripts": {
    "gcp-build": "next build"
  },
  "engines": {
    "node": "18.x"
  }
}
```

#### 4. 環境變數未載入

**問題**：應用程式無法讀取環境變數

**解決方法**：

Cloud Run：

```bash
# 確認環境變數已設定
gcloud run services describe my-nextjs-app \
  --region asia-east1 \
  --format "value(spec.template.spec.containers[0].env)"

# 重新部署
gcloud run services update my-nextjs-app \
  --region asia-east1 \
  --set-env-vars "KEY=value"
```

App Engine：

```yaml
# app.yaml
env_variables:
  DATABASE_URL: "your-value"
```

Next.js 設定：

```javascript
// next.config.js
module.exports = {
  // 公開環境變數（前端可存取）
  env: {
    CUSTOM_KEY: process.env.CUSTOM_KEY,
  },
}
```

#### 5. 靜態資源 404

**問題**：圖片、CSS、JS 檔案無法載入

**解決方法**：

確保 Dockerfile 複製所有必要檔案：

```dockerfile
# 複製 public 目錄
COPY --from=builder /app/public ./public

# 複製靜態資源
COPY --from=builder /app/.next/static ./.next/static
```

Next.js 設定：

```javascript
// next.config.js
module.exports = {
  // 設定 asset prefix（如果使用 CDN）
  assetPrefix: process.env.NODE_ENV === 'production'
    ? 'https://cdn.example.com'
    : '',
}
```

#### 6. CORS 錯誤

**問題**：API 請求被 CORS 政策阻擋

**解決方法**：

```javascript
// pages/api/[...].js
export default function handler(req, res) {
  // 設定 CORS 標頭
  res.setHeader('Access-Control-Allow-Origin', '*')
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS')
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization')

  // 處理 preflight 請求
  if (req.method === 'OPTIONS') {
    res.status(200).end()
    return
  }

  // 你的 API 邏輯
}
```

或使用中介軟體：

```javascript
// middleware.js
import { NextResponse } from 'next/server'

export function middleware(request) {
  const response = NextResponse.next()

  response.headers.set('Access-Control-Allow-Origin', '*')
  response.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS')

  return response
}
```

#### 7. Cloud Run: 請求逾時

**錯誤**：`Request timeout`

**解決方法**：

```bash
# 增加逾時時間（最多 60 分鐘）
gcloud run services update my-nextjs-app \
  --region asia-east1 \
  --timeout 300
```

優化長時間執行的任務：

```javascript
// 使用背景任務或佇列
// 考慮使用 Cloud Tasks 或 Pub/Sub
```

### 除錯技巧

#### 1. 本地測試 Docker 映像

```bash
# 建置映像
docker build -t my-nextjs-app .

# 執行容器
docker run -p 8080:8080 \
  -e PORT=8080 \
  -e DATABASE_URL=postgresql://... \
  my-nextjs-app

# 測試
curl http://localhost:8080
```

#### 2. 查看即時日誌

```bash
# Cloud Run
gcloud logging tail "resource.type=cloud_run_revision AND resource.labels.service_name=my-nextjs-app"

# App Engine
gcloud app logs tail -s default
```

#### 3. SSH 連線到容器（除錯）

```bash
# Cloud Run（需要啟用 Cloud Run Admin API）
gcloud run services proxy my-nextjs-app --region asia-east1

# 在另一個終端
docker exec -it [CONTAINER_ID] /bin/sh
```

#### 4. 效能分析

```bash
# 啟用 Cloud Profiler
npm install @google-cloud/profiler

# 在應用程式中
require('@google-cloud/profiler').start({
  serviceContext: {
    service: 'my-nextjs-app',
    version: '1.0.0',
  },
})
```

## 成本優化建議

### 1. Cloud Run

- 使用最小實例數 0（避免持續計費）
- 設定適當的 CPU 和記憶體配置
- 啟用請求逾時
- 使用 Cloud CDN 減少請求數

### 2. App Engine

- 使用標準環境（比彈性環境便宜）
- 設定合理的自動擴展參數
- 刪除舊版本
- 使用流量分配進行 A/B 測試

### 3. 通用建議

- 使用 Cloud Storage 儲存靜態資源
- 啟用快取機制
- 監控使用量並設定預算警示
- 使用免費額度進行測試

## 總結

### Cloud Run

✅ **優勢**：
- 容器化、高度客製化
- 冷啟動快、自動擴展到零
- 按請求計費、成本效益高
- 適合現代化應用

⚠️ **注意事項**：
- 需要 Docker 知識
- 冷啟動可能有延遲
- 需要注意請求逾時

### App Engine

✅ **優勢**：
- 簡單部署、零配置
- 自動擴展、負載平衡
- 整合 GCP 服務
- 適合傳統應用

⚠️ **注意事項**：
- 至少一個實例持續運行（成本較高）
- 自訂程度較低
- 彈性環境啟動較慢

🎯 **推薦選擇**：
- **Cloud Run**：適合大多數 Next.js 專案
- **App Engine**：適合需要簡單部署的專案

下一章我們將學習如何部署到 Cloudflare Pages。
