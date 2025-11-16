# 17. 部署到 Google Cloud Platform (GCP)

## GCP 簡介與服務選擇

Google Cloud Platform (GCP) 是 Google 提供的雲端運算服務平台，提供多種方式來部署 Nuxt.js 應用。

### GCP 主要優勢

- ✅ **強大的基礎設施**：Google 級別的全球網路
- ✅ **豐富的服務選擇**：從簡單到複雜的各種部署方案
- ✅ **良好的擴展性**：自動擴展和負載均衡
- ✅ **整合 Google 服務**：Firebase、BigQuery、AI/ML 等
- ✅ **細緻的成本控制**：按使用量付費
- ✅ **企業級安全**：符合多種合規標準

### Nuxt.js 部署選項比較

| 服務 | 適用場景 | 難度 | 費用 | 推薦指數 |
|------|---------|------|------|---------|
| **Cloud Run** | SSR、API、容器化應用 | ⭐⭐ | 💰 | ⭐⭐⭐⭐⭐ |
| **App Engine** | 傳統 Web 應用 | ⭐⭐⭐ | 💰💰 | ⭐⭐⭐⭐ |
| **Firebase Hosting** | 靜態網站、SSG | ⭐ | 💰 | ⭐⭐⭐ |
| **Compute Engine** | 完全控制、自定義環境 | ⭐⭐⭐⭐ | 💰💰💰 | ⭐⭐ |
| **Cloud Storage + CDN** | 純靜態網站 | ⭐⭐ | 💰 | ⭐⭐⭐ |

---

## Cloud Run 部署（推薦）

Cloud Run 是 GCP 的無服務器容器平台，非常適合部署 Nuxt.js SSR 應用。

### Cloud Run 優勢

- **自動擴展**：從 0 擴展到數千個實例
- **按使用付費**：沒有流量時不收費
- **容器化**：使用 Docker，環境一致
- **快速部署**：幾秒鐘內部署新版本
- **全球分佈**：支援多區域部署

### 前置準備

#### 1. 安裝 Google Cloud SDK

```bash
# macOS (使用 Homebrew)
brew install --cask google-cloud-sdk

# Ubuntu/Debian
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install google-cloud-cli

# Windows
# 下載並安裝: https://cloud.google.com/sdk/docs/install
```

#### 2. 初始化 gcloud

```bash
# 登入 Google 帳號
gcloud auth login

# 設定項目
gcloud config set project YOUR_PROJECT_ID

# 查看當前配置
gcloud config list
```

#### 3. 啟用必要的 API

```bash
# 啟用 Cloud Run API
gcloud services enable run.googleapis.com

# 啟用 Cloud Build API
gcloud services enable cloudbuild.googleapis.com

# 啟用 Artifact Registry API
gcloud services enable artifactregistry.googleapis.com
```

### 建立 Dockerfile

在項目根目錄建立 `Dockerfile`：

```dockerfile
# Dockerfile
# 使用官方 Node.js 18 映像作為基礎
FROM node:18-alpine AS base

# 安裝依賴階段
FROM base AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# 構建階段
FROM base AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 生產階段
FROM base AS production
WORKDIR /app

# 複製依賴
COPY --from=dependencies /app/node_modules ./node_modules

# 複製構建產物
COPY --from=build /app/.output ./.output
COPY --from=build /app/package*.json ./

# 設定環境變數
ENV NODE_ENV=production
ENV PORT=8080

# 暴露端口（Cloud Run 預設使用 8080）
EXPOSE 8080

# 啟動應用
CMD ["node", ".output/server/index.mjs"]
```

### 優化的多階段 Dockerfile

```dockerfile
# 更優化的版本
FROM node:18-alpine AS base

# 依賴階段
FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production && \
    npm cache clean --force

# 構建階段
FROM base AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .

# 設定構建時環境變數
ARG NUXT_PUBLIC_API_BASE
ENV NUXT_PUBLIC_API_BASE=$NUXT_PUBLIC_API_BASE

RUN npm run build

# 運行階段
FROM base AS runner
WORKDIR /app

# 創建非 root 用戶
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nuxtjs

# 複製文件
COPY --from=deps --chown=nuxtjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nuxtjs:nodejs /app/.output ./.output
COPY --from=builder --chown=nuxtjs:nodejs /app/package.json ./

# 設定用戶
USER nuxtjs

# 環境變數
ENV NODE_ENV=production
ENV PORT=8080

EXPOSE 8080

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
  CMD node -e "require('http').get('http://localhost:8080', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

CMD ["node", ".output/server/index.mjs"]
```

### .dockerignore 文件

```
# .dockerignore
node_modules
npm-debug.log
.nuxt
.output
.git
.gitignore
.env
.env.*
dist
.DS_Store
README.md
.vscode
.idea
coverage
*.log
.cache
```

### Nuxt 配置

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 確保使用 node-server preset
  nitro: {
    preset: 'node-server'
  },

  // 運行時配置
  runtimeConfig: {
    // 私有環境變數
    apiSecret: process.env.API_SECRET,
    databaseUrl: process.env.DATABASE_URL,

    public: {
      // 公開環境變數
      apiBase: process.env.NUXT_PUBLIC_API_BASE || 'https://api.example.com',
      siteUrl: process.env.NUXT_PUBLIC_SITE_URL || 'https://example.com'
    }
  },

  // 生產環境優化
  app: {
    head: {
      charset: 'utf-8',
      viewport: 'width=device-width, initial-scale=1',
      meta: [
        { name: 'robots', content: 'index, follow' }
      ]
    }
  },

  // 路由規則
  routeRules: {
    '/': { prerender: true },
    '/api/**': { cors: true }
  }
})
```

### 部署到 Cloud Run

#### 方法 1：使用 gcloud 命令（推薦）

```bash
# 構建並部署（一步完成）
gcloud run deploy my-nuxt-app \
  --source . \
  --region asia-east1 \
  --platform managed \
  --allow-unauthenticated \
  --set-env-vars "NUXT_PUBLIC_API_BASE=https://api.example.com" \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \
  --max-instances 10 \
  --timeout 60 \
  --port 8080

# 部署成功後會顯示：
# Service [my-nuxt-app] revision [my-nuxt-app-00001-abc] has been deployed
# and is serving 100 percent of traffic.
# Service URL: https://my-nuxt-app-xxxxxx-de.a.run.app
```

#### 方法 2：使用 Cloud Build + Artifact Registry

**步驟 1：建立 Artifact Registry**

```bash
# 建立 Docker 倉庫
gcloud artifacts repositories create my-nuxt-app \
  --repository-format=docker \
  --location=asia-east1 \
  --description="Nuxt.js application"

# 配置 Docker 認證
gcloud auth configure-docker asia-east1-docker.pkg.dev
```

**步驟 2：構建並推送映像**

```bash
# 設定變數
PROJECT_ID=$(gcloud config get-value project)
REGION=asia-east1
REPO_NAME=my-nuxt-app
IMAGE_NAME=nuxt-app

# 構建 Docker 映像
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:latest .

# 推送到 Artifact Registry
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:latest
```

**步驟 3：部署到 Cloud Run**

```bash
gcloud run deploy my-nuxt-app \
  --image ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}/${IMAGE_NAME}:latest \
  --region ${REGION} \
  --platform managed \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1
```

### 使用 cloudbuild.yaml 自動化

建立 `cloudbuild.yaml`：

```yaml
# cloudbuild.yaml
steps:
  # 構建 Docker 映像
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:$COMMIT_SHA'
      - '-t'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:latest'
      - '.'

  # 推送映像到 Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:$COMMIT_SHA'

  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:latest'

  # 部署到 Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'my-nuxt-app'
      - '--image'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:$COMMIT_SHA'
      - '--region'
      - 'asia-east1'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'

# 設定映像推送到 Artifact Registry
images:
  - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:$COMMIT_SHA'
  - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:latest'

# 構建選項
options:
  machineType: 'N1_HIGHCPU_8'
  logging: CLOUD_LOGGING_ONLY
```

執行構建：

```bash
gcloud builds submit --config cloudbuild.yaml
```

---

## App Engine 部署

App Engine 是 GCP 的傳統 PaaS 平台，適合不想處理容器的團隊。

### App Engine 配置

建立 `app.yaml`：

```yaml
# app.yaml
runtime: nodejs18

# 實例配置
instance_class: F2
automatic_scaling:
  min_instances: 0
  max_instances: 10
  target_cpu_utilization: 0.75

# 環境變數
env_variables:
  NODE_ENV: 'production'
  NUXT_PUBLIC_API_BASE: 'https://api.example.com'

# 處理器配置
handlers:
  # 靜態資源快取
  - url: /_nuxt/.*
    static_dir: .output/public/_nuxt
    secure: always
    http_headers:
      Cache-Control: 'public, max-age=31536000, immutable'

  # 所有其他請求
  - url: /.*
    script: auto
    secure: always

# 健康檢查
liveness_check:
  path: '/'
  check_interval_sec: 30
  timeout_sec: 4
  failure_threshold: 2

readiness_check:
  path: '/'
  check_interval_sec: 5
  timeout_sec: 4
  failure_threshold: 2
  app_start_timeout_sec: 300
```

### package.json 配置

```json
{
  "name": "my-nuxt-app",
  "version": "1.0.0",
  "scripts": {
    "dev": "nuxt dev",
    "build": "nuxt build",
    "start": "node .output/server/index.mjs",
    "preview": "nuxt preview",
    "gcp-build": "nuxt build"
  },
  "engines": {
    "node": "18.x"
  }
}
```

### 部署到 App Engine

```bash
# 初始化 App Engine（首次）
gcloud app create --region=asia-east1

# 部署應用
gcloud app deploy

# 部署到特定版本
gcloud app deploy --version v1 --no-promote

# 查看應用
gcloud app browse

# 查看日誌
gcloud app logs tail -s default

# 查看實例
gcloud app instances list
```

### 多版本部署與流量分配

```bash
# 部署新版本（不切換流量）
gcloud app deploy --version v2 --no-promote

# 查看所有版本
gcloud app versions list

# 分配流量（金絲雀部署）
gcloud app services set-traffic default \
  --splits v1=90,v2=10

# 完全切換到新版本
gcloud app services set-traffic default \
  --splits v2=100

# 刪除舊版本
gcloud app versions delete v1
```

---

## 環境變數與 Secret 管理

### 使用 Secret Manager

#### 1. 建立 Secret

```bash
# 啟用 Secret Manager API
gcloud services enable secretmanager.googleapis.com

# 建立 secret
echo -n "your-api-secret-key" | \
  gcloud secrets create api-secret --data-file=-

# 建立資料庫 URL secret
echo -n "postgresql://user:pass@host:5432/db" | \
  gcloud secrets create database-url --data-file=-

# 查看 secrets
gcloud secrets list
```

#### 2. 授予權限

```bash
# 授予 Cloud Run 服務帳號權限
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")

gcloud secrets add-iam-policy-binding api-secret \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

#### 3. 在 Cloud Run 中使用 Secrets

```bash
gcloud run deploy my-nuxt-app \
  --source . \
  --region asia-east1 \
  --set-secrets="API_SECRET=api-secret:latest,DATABASE_URL=database-url:latest" \
  --allow-unauthenticated
```

### 使用 .env 文件（開發環境）

```bash
# .env
NODE_ENV=production
NUXT_PUBLIC_API_BASE=https://api.example.com
NUXT_PUBLIC_SITE_URL=https://example.com
API_SECRET=your-secret-key
DATABASE_URL=postgresql://localhost:5432/mydb
```

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  runtimeConfig: {
    apiSecret: process.env.API_SECRET,
    databaseUrl: process.env.DATABASE_URL,

    public: {
      apiBase: process.env.NUXT_PUBLIC_API_BASE,
      siteUrl: process.env.NUXT_PUBLIC_SITE_URL
    }
  }
})
```

---

## 自定義域名與 SSL

### Cloud Run 自定義域名

#### 1. 驗證域名所有權

```bash
# 添加自定義域名映射
gcloud run domain-mappings create \
  --service my-nuxt-app \
  --domain www.example.com \
  --region asia-east1
```

#### 2. 配置 DNS

系統會提示你添加 DNS 記錄：

```
Type: CNAME
Name: www
Value: ghs.googlehosted.com
```

或使用 A 記錄：

```
Type: A
Name: @
Value:
  - 216.239.32.21
  - 216.239.34.21
  - 216.239.36.21
  - 216.239.38.21
```

#### 3. SSL 證書

Google 會自動為你的域名配置 SSL 證書（Let's Encrypt）。

### Cloud Load Balancer + Cloud Run

對於更複雜的場景，使用 Cloud Load Balancer：

```bash
# 1. 建立 NEG (Network Endpoint Group)
gcloud compute network-endpoint-groups create nuxt-neg \
  --region=asia-east1 \
  --network-endpoint-type=serverless \
  --cloud-run-service=my-nuxt-app

# 2. 建立後端服務
gcloud compute backend-services create nuxt-backend \
  --global

# 3. 添加後端
gcloud compute backend-services add-backend nuxt-backend \
  --global \
  --network-endpoint-group=nuxt-neg \
  --network-endpoint-group-region=asia-east1

# 4. 建立 URL Map
gcloud compute url-maps create nuxt-url-map \
  --default-service nuxt-backend

# 5. 建立 SSL 證書
gcloud compute ssl-certificates create nuxt-ssl-cert \
  --domains=example.com,www.example.com \
  --global

# 6. 建立 HTTPS Proxy
gcloud compute target-https-proxies create nuxt-https-proxy \
  --url-map=nuxt-url-map \
  --ssl-certificates=nuxt-ssl-cert

# 7. 建立轉發規則
gcloud compute forwarding-rules create nuxt-https-rule \
  --address=nuxt-ip \
  --global \
  --target-https-proxy=nuxt-https-proxy \
  --ports=443
```

---

## CI/CD 整合

### 使用 Cloud Build 觸發器

#### 1. 連接 GitHub/GitLab

在 Cloud Console 中：
1. 訪問 Cloud Build → Triggers
2. 點擊 "Connect Repository"
3. 選擇 GitHub 或 GitLab
4. 授權並選擇倉庫

#### 2. 建立觸發器

```bash
# 使用 gcloud 建立觸發器
gcloud builds triggers create github \
  --repo-name=my-nuxt-app \
  --repo-owner=your-username \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml
```

#### 3. 完整的 CI/CD cloudbuild.yaml

```yaml
# cloudbuild.yaml
steps:
  # 安裝依賴
  - name: 'node:18'
    entrypoint: npm
    args: ['ci']

  # 運行測試
  - name: 'node:18'
    entrypoint: npm
    args: ['run', 'test']
    env:
      - 'NODE_ENV=test'

  # 運行 Lint
  - name: 'node:18'
    entrypoint: npm
    args: ['run', 'lint']

  # 構建應用
  - name: 'node:18'
    entrypoint: npm
    args: ['run', 'build']
    env:
      - 'NODE_ENV=production'

  # 構建 Docker 映像
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:$SHORT_SHA'
      - '-t'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:latest'
      - '--cache-from'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:latest'
      - '.'

  # 推送映像
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - '--all-tags'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app'

  # 部署到 Cloud Run (Staging)
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'my-nuxt-app-staging'
      - '--image'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:$SHORT_SHA'
      - '--region'
      - 'asia-east1'
      - '--platform'
      - 'managed'
      - '--tag'
      - 'staging'
      - '--no-traffic'

  # 運行冒煙測試
  - name: 'node:18'
    entrypoint: npm
    args: ['run', 'test:e2e']
    env:
      - 'TEST_URL=https://staging---my-nuxt-app-staging-xxxxxx-de.a.run.app'

  # 部署到生產環境
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'my-nuxt-app'
      - '--image'
      - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:$SHORT_SHA'
      - '--region'
      - 'asia-east1'
      - '--platform'
      - 'managed'
      - '--allow-unauthenticated'

images:
  - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:$SHORT_SHA'
  - 'asia-east1-docker.pkg.dev/$PROJECT_ID/my-nuxt-app/nuxt-app:latest'

options:
  machineType: 'N1_HIGHCPU_8'
  timeout: '1800s'
```

### GitHub Actions 整合

```yaml
# .github/workflows/deploy-gcp.yml
name: Deploy to GCP

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  REGION: asia-east1
  SERVICE_NAME: my-nuxt-app

jobs:
  deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write

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
        run: npm run test

      - name: Run lint
        run: npm run lint

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.WIF_SERVICE_ACCOUNT }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Authorize Docker
        run: |
          gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev

      - name: Build and Push Container
        run: |
          docker build -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/my-nuxt-app/nuxt-app:${{ github.sha }} .
          docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/my-nuxt-app/nuxt-app:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy ${{ env.SERVICE_NAME }} \
            --image ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/my-nuxt-app/nuxt-app:${{ github.sha }} \
            --region ${{ env.REGION }} \
            --platform managed \
            --allow-unauthenticated

      - name: Show Service URL
        run: |
          echo "Service URL: $(gcloud run services describe ${{ env.SERVICE_NAME }} --region ${{ env.REGION }} --format 'value(status.url)')"
```

---

## 成本估算

### Cloud Run 定價（2024）

**請求費用：**
```
前 200 萬次請求/月：免費
後續請求：$0.40 / 100 萬次請求
```

**CPU 費用：**
```
vCPU: $0.00002400 / vCPU-秒
記憶體: $0.00000250 / GiB-秒
```

**範例計算（中型網站）：**

```
假設：
- 100 萬次請求/月
- 平均請求時間：200ms
- CPU: 1 vCPU
- 記憶體: 512 MB

計算：
請求費用：免費（在免費額度內）
CPU 費用：1,000,000 × 0.2s × $0.00002400 = $4.80
記憶體費用：1,000,000 × 0.2s × 0.5GB × $0.00000250 = $0.25

總計：約 $5 /月
```

### App Engine 定價

**Standard 環境：**
```
實例時間：
- F1 (256MB, 600MHz): $0.05 /實例小時
- F2 (512MB, 1.2GHz): $0.10 /實例小時
- F4 (1GB, 2.4GHz): $0.20 /實例小時

範例：
- F2 實例，平均 2 個實例運行
- $0.10 × 2 × 24 × 30 = $144 /月
```

### Artifact Registry 定價

```
儲存：$0.10 / GB /月
網路流出：
- 前 1GB/月：免費
- 後續：依地區而定（約 $0.12/GB）
```

### 成本優化建議

1. **使用 Cloud Run 而非 App Engine**
   - 自動擴展到零
   - 按實際使用付費

2. **設定最小實例數為 0**
   ```bash
   gcloud run deploy my-nuxt-app --min-instances 0
   ```

3. **優化容器大小**
   - 使用 Alpine Linux
   - 多階段構建
   - 清理不必要的文件

4. **使用 Cloud CDN**
   ```bash
   gcloud compute backend-services update nuxt-backend \
     --enable-cdn --global
   ```

5. **設定預算警報**
   ```bash
   gcloud billing budgets create \
     --billing-account=BILLING_ACCOUNT_ID \
     --display-name="Nuxt App Budget" \
     --budget-amount=100 \
     --threshold-rule=percent=50 \
     --threshold-rule=percent=90
   ```

---

## 完整部署步驟

### 準備工作

```bash
# 1. 建立 GCP 項目
gcloud projects create my-nuxt-app-project
gcloud config set project my-nuxt-app-project

# 2. 啟用必要的 API
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  secretmanager.googleapis.com

# 3. 建立 Artifact Registry
gcloud artifacts repositories create my-nuxt-app \
  --repository-format=docker \
  --location=asia-east1
```

### 項目配置

```bash
# 4. 建立 Dockerfile（見上面的範例）
touch Dockerfile
touch .dockerignore

# 5. 配置 nuxt.config.ts
# 確保 preset 設為 'node-server'
```

### 首次部署

```bash
# 6. 本地測試構建
docker build -t my-nuxt-app:test .
docker run -p 3000:8080 my-nuxt-app:test

# 7. 部署到 Cloud Run
gcloud run deploy my-nuxt-app \
  --source . \
  --region asia-east1 \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --min-instances 0 \
  --max-instances 10 \
  --port 8080

# 8. 訪問應用
gcloud run services describe my-nuxt-app \
  --region asia-east1 \
  --format 'value(status.url)'
```

### 設定 CI/CD

```bash
# 9. 建立 cloudbuild.yaml
# 10. 連接 GitHub 倉庫
# 11. 建立觸發器
gcloud builds triggers create github \
  --repo-name=my-nuxt-app \
  --repo-owner=your-username \
  --branch-pattern="^main$" \
  --build-config=cloudbuild.yaml
```

### 配置域名

```bash
# 12. 添加自定義域名
gcloud run domain-mappings create \
  --service my-nuxt-app \
  --domain www.example.com \
  --region asia-east1

# 13. 配置 DNS（按提示操作）
```

---

## 疑難排解

### 問題 1：容器啟動失敗

**錯誤訊息：**
```
Container failed to start. Failed to start and then listen on the port defined by the PORT environment variable.
```

**解決方案：**

```typescript
// 確保應用監聽正確的端口
// nuxt.config.ts
export default defineNuxtConfig({
  nitro: {
    preset: 'node-server'
  }
})
```

```dockerfile
# Dockerfile 中設定 PORT
ENV PORT=8080
EXPOSE 8080
```

### 問題 2：記憶體不足

**錯誤：** Container memory limit exceeded

**解決方案：**

```bash
# 增加記憶體配置
gcloud run deploy my-nuxt-app \
  --memory 1Gi \
  --cpu 2
```

或優化應用：

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  experimental: {
    payloadExtraction: true
  },

  nitro: {
    minify: true,
    compressPublicAssets: true
  }
})
```

### 問題 3：構建超時

**錯誤：** Build timeout

**解決方案：**

```yaml
# cloudbuild.yaml
options:
  machineType: 'N1_HIGHCPU_8'  # 使用更強大的機器
  timeout: '1800s'  # 增加超時時間
```

### 問題 4：環境變數未生效

**問題：** process.env 讀不到值

**解決方案：**

```typescript
// ❌ 錯誤
const apiKey = process.env.API_KEY

// ✅ 正確
const config = useRuntimeConfig()
const apiKey = config.apiKey
```

### 問題 5：CORS 錯誤

**解決方案：**

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/api/**': {
      cors: true,
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE',
        'Access-Control-Allow-Headers': 'Content-Type'
      }
    }
  }
})
```

### 查看日誌

```bash
# Cloud Run 日誌
gcloud run services logs read my-nuxt-app \
  --region asia-east1 \
  --limit 50

# Cloud Build 日誌
gcloud builds log BUILD_ID

# 即時查看日誌
gcloud run services logs tail my-nuxt-app \
  --region asia-east1
```

---

## 總結

### GCP 優點

✅ 強大的基礎設施和全球網路
✅ 豐富的服務選擇
✅ Cloud Run 非常適合 Nuxt.js SSR
✅ 良好的擴展性和效能
✅ 整合 Google 生態系統
✅ 企業級安全和合規

### GCP 缺點

❌ 學習曲線較陡
❌ 配置比 Vercel 複雜
❌ 費用計算較複雜
❌ 文檔有時不夠友好

### 推薦使用場景

1. **企業級應用** - 需要強大基礎設施
2. **高流量網站** - 成本比其他平台更低
3. **需要複雜整合** - BigQuery、AI/ML 等
4. **已使用 GCP** - 整合現有服務

### 最佳實踐

1. 使用 Cloud Run 而非 App Engine（更靈活、更便宜）
2. 使用 Secret Manager 管理敏感資訊
3. 設定 CI/CD 自動化部署
4. 使用 Cloud CDN 加速靜態資源
5. 配置預算警報避免超支
6. 定期檢查日誌和監控

---

## 參考資源

- [Cloud Run 文檔](https://cloud.google.com/run/docs)
- [App Engine 文檔](https://cloud.google.com/appengine/docs)
- [Cloud Build 文檔](https://cloud.google.com/build/docs)
- [Secret Manager](https://cloud.google.com/secret-manager/docs)
- [GCP 定價計算器](https://cloud.google.com/products/calculator)
- [Nuxt.js 部署文檔](https://nuxt.com/docs/getting-started/deployment)
