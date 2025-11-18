# Chapter 57: Docker 容器化

## 概述

Docker 容器化使應用部署更加簡單和一致。本章將介紹如何為 Go 應用創建高效的 Docker 鏡像，並與 Node.js 的 Docker 實踐進行對比。

## Go vs Node.js Docker 對比

### Node.js Dockerfile
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

**鏡像大小**：約 150-200 MB

### Go Dockerfile（基礎）
```dockerfile
FROM golang:1.21-alpine
WORKDIR /app
COPY . .
RUN go build -o main .
EXPOSE 8080
CMD ["./main"]
```

**鏡像大小**：約 300-400 MB

### Go Dockerfile（優化）
```dockerfile
FROM alpine:latest
COPY main /
EXPOSE 8080
CMD ["/main"]
```

**鏡像大小**：約 10-20 MB

## 多階段構建（Multi-Stage Build）

多階段構建是 Go Docker 化的最佳實踐，可以大幅減小最終鏡像大小。

### 基本多階段構建

```dockerfile
# 階段 1: 構建
FROM golang:1.21-alpine AS builder

# 安裝構建依賴
RUN apk add --no-cache git make

# 設置工作目錄
WORKDIR /app

# 複製 go mod 文件
COPY go.mod go.sum ./

# 下載依賴
RUN go mod download

# 複製源代碼
COPY . .

# 構建應用
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags="-s -w" -o main ./cmd/api

# 階段 2: 運行
FROM alpine:latest

# 安裝 ca-certificates（HTTPS 請求需要）
RUN apk --no-cache add ca-certificates tzdata

# 創建非 root 用戶
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

WORKDIR /home/appuser

# 從構建階段複製二進制文件
COPY --from=builder /app/main .

# 複製配置文件（如果需要）
COPY --from=builder /app/config ./config

# 切換到非 root 用戶
USER appuser

EXPOSE 8080

CMD ["./main"]
```

### 優化的多階段構建

```dockerfile
# syntax=docker/dockerfile:1

##
## 階段 1: 構建環境
##
FROM golang:1.21-alpine AS builder

# 安裝必要工具
RUN apk add --no-cache \
    git \
    make \
    ca-certificates

WORKDIR /build

# 利用 Docker 層緩存，先複製依賴文件
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# 複製源代碼
COPY . .

# 構建參數
ARG VERSION=dev
ARG BUILD_TIME
ARG GIT_COMMIT

# 編譯
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-s -w \
    -X main.Version=${VERSION} \
    -X main.BuildTime=${BUILD_TIME} \
    -X main.GitCommit=${GIT_COMMIT}" \
    -a -installsuffix cgo \
    -o app ./cmd/api

##
## 階段 2: 運行環境
##
FROM scratch

# 從 builder 複製 CA 證書
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 從 builder 複製時區信息
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# 複製二進制文件
COPY --from=builder /build/app /app

# 複製配置文件
COPY configs /configs

# 暴露端口
EXPOSE 8080

# 設置入口點
ENTRYPOINT ["/app"]
```

## 完整的 Docker 化項目

### 項目結構

```
my-go-app/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   └── ...
├── configs/
│   ├── config.yaml
│   └── config.prod.yaml
├── Dockerfile
├── Dockerfile.dev
├── .dockerignore
├── docker-compose.yml
├── Makefile
└── go.mod
```

### 1. Dockerfile（生產環境）

```dockerfile
# Dockerfile

# 構建階段
FROM golang:1.21-alpine AS builder

ARG VERSION=dev
ARG BUILD_TIME
ARG GIT_COMMIT

WORKDIR /build

# 依賴管理
COPY go.mod go.sum ./
RUN go mod download

# 構建
COPY . .
RUN CGO_ENABLED=0 go build \
    -ldflags="-s -w -X main.Version=${VERSION} -X main.BuildTime=${BUILD_TIME} -X main.GitCommit=${GIT_COMMIT}" \
    -o app ./cmd/api

# 運行階段
FROM alpine:latest

RUN apk --no-cache add ca-certificates tzdata

# 創建非 root 用戶
RUN addgroup -g 1000 app && \
    adduser -D -u 1000 -G app app

WORKDIR /home/app

COPY --from=builder /build/app .
COPY configs ./configs

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

USER app

EXPOSE 8080

CMD ["./app"]
```

### 2. Dockerfile.dev（開發環境）

```dockerfile
# Dockerfile.dev

FROM golang:1.21-alpine

# 安裝開發工具
RUN apk add --no-cache \
    git \
    make \
    curl

# 安裝 air（熱重載工具）
RUN go install github.com/cosmtrek/air@latest

WORKDIR /app

# 複製依賴文件
COPY go.mod go.sum ./
RUN go mod download

# 複製源代碼
COPY . .

EXPOSE 8080

# 使用 air 進行熱重載
CMD ["air", "-c", ".air.toml"]
```

### 3. .dockerignore

```
# .dockerignore

# Git
.git
.gitignore

# IDEs
.idea
.vscode
*.swp
*.swo

# Build artifacts
bin/
dist/
*.exe
*.test

# Dependencies
vendor/

# Logs
*.log

# OS
.DS_Store
Thumbs.db

# Docker
Dockerfile*
docker-compose*.yml

# Documentation
README.md
docs/

# CI/CD
.github/
.gitlab-ci.yml
```

### 4. docker-compose.yml

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        VERSION: ${VERSION:-dev}
        BUILD_TIME: ${BUILD_TIME}
        GIT_COMMIT: ${GIT_COMMIT}
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/mydb?sslmode=disable
      - REDIS_URL=redis:6379
      - ENV=production
    depends_on:
      - postgres
      - redis
    networks:
      - app-network
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  postgres-data:
  redis-data:

networks:
  app-network:
    driver: bridge
```

### 5. docker-compose.dev.yml（開發環境）

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - go-modules:/go/pkg/mod
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:postgres@postgres:5432/mydb?sslmode=disable
      - REDIS_URL=redis:6379
      - ENV=development
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres-dev-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres-dev-data:
  go-modules:
```

### 6. Makefile

```makefile
# Makefile

APP_NAME=myapp
VERSION=$(shell git describe --tags --always --dirty)
BUILD_TIME=$(shell date -u '+%Y-%m-%d_%H:%M:%S')
GIT_COMMIT=$(shell git rev-parse HEAD)

.PHONY: docker-build docker-run docker-push dev-up dev-down

# 構建 Docker 鏡像
docker-build:
	docker build \
		--build-arg VERSION=$(VERSION) \
		--build-arg BUILD_TIME=$(BUILD_TIME) \
		--build-arg GIT_COMMIT=$(GIT_COMMIT) \
		-t $(APP_NAME):$(VERSION) \
		-t $(APP_NAME):latest \
		.

# 運行 Docker 容器
docker-run:
	docker run -p 8080:8080 $(APP_NAME):latest

# 推送到 Docker Hub
docker-push:
	docker push $(APP_NAME):$(VERSION)
	docker push $(APP_NAME):latest

# 啟動開發環境
dev-up:
	docker-compose -f docker-compose.dev.yml up -d

# 停止開發環境
dev-down:
	docker-compose -f docker-compose.dev.yml down

# 查看日誌
dev-logs:
	docker-compose -f docker-compose.dev.yml logs -f app

# 進入容器
dev-shell:
	docker-compose -f docker-compose.dev.yml exec app sh

# 生產環境
prod-up:
	docker-compose up -d

prod-down:
	docker-compose down

# 清理
clean:
	docker-compose down -v
	docker system prune -f
```

## 優化技巧

### 1. 使用 BuildKit

```bash
# 啟用 BuildKit
export DOCKER_BUILDKIT=1

# 或在 Dockerfile 頂部添加
# syntax=docker/dockerfile:1
```

### 2. 利用層緩存

```dockerfile
# 好的做法：先複製依賴文件
COPY go.mod go.sum ./
RUN go mod download

# 然後複製源代碼
COPY . .

# 避免：一次性複製所有文件
# COPY . .
# RUN go mod download
```

### 3. 使用 scratch 或 distroless

```dockerfile
# 使用 scratch（最小）
FROM scratch
COPY --from=builder /app/main /main
CMD ["/main"]

# 使用 distroless（包含基本工具）
FROM gcr.io/distroless/static-debian11
COPY --from=builder /app/main /main
CMD ["/main"]
```

### 4. 多架構構建

```bash
# 創建 builder
docker buildx create --name mybuilder --use

# 多架構構建並推送
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:latest \
  --push \
  .
```

## Node.js Docker 最佳實踐對比

### Node.js 多階段構建

```dockerfile
# 構建階段
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

# 運行階段
FROM node:18-alpine

WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules
COPY . .

USER node

EXPOSE 3000

CMD ["node", "app.js"]
```

### 對比總結

| 特性 | Go | Node.js |
|------|-----|---------|
| 基礎鏡像大小 | Alpine: ~5 MB | Alpine: ~40 MB |
| 最終鏡像大小 | 10-30 MB | 100-150 MB |
| 構建時間 | 快 | 中等 |
| 運行時依賴 | 無 | Node.js 運行時 |
| 熱重載 | 需要額外工具 | 原生支持 |

## 安全最佳實踐

### 1. 使用非 root 用戶

```dockerfile
# 創建用戶
RUN addgroup -g 1000 app && \
    adduser -D -u 1000 -G app app

# 切換用戶
USER app
```

### 2. 掃描漏洞

```bash
# 使用 Trivy 掃描
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:latest

# 使用 Docker scan
docker scan myapp:latest
```

### 3. 使用 .dockerignore

確保不包含敏感文件：
```
.env
*.key
*.pem
secrets/
```

### 4. 設置資源限制

```yaml
# docker-compose.yml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

## CI/CD 集成

### GitHub Actions

```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: myorg/myapp

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - build
  - push

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest

push:
  stage: push
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
```

## 重點總結

### Go Docker 化優勢

1. **鏡像小**
   - 使用多階段構建
   - 最終鏡像可小至 10 MB

2. **安全性高**
   - 無運行時依賴
   - 可使用 scratch 鏡像

3. **啟動快**
   - 靜態二進制
   - 無需解釋器

### 最佳實踐

1. **使用多階段構建**
   - 分離構建和運行環境
   - 減小鏡像大小

2. **優化層緩存**
   - 先複製依賴文件
   - 充分利用 Docker 層緩存

3. **安全考慮**
   - 使用非 root 用戶
   - 定期掃描漏洞
   - 不包含敏感文件

4. **健康檢查**
   - 添加 HEALTHCHECK
   - 實現 /health 端點

## 練習題

### 練習 1: 優化 Dockerfile

優化以下 Dockerfile，減小鏡像大小：
```dockerfile
FROM golang:1.21
WORKDIR /app
COPY . .
RUN go build -o main
CMD ["./main"]
```

### 練習 2: 添加健康檢查

為 Docker 容器添加健康檢查功能。

### 練習 3: 多階段構建

實現包含測試階段的多階段構建：
1. 依賴階段
2. 測試階段
3. 構建階段
4. 運行階段

### 練習 4: Docker Compose 微服務

創建包含以下服務的 docker-compose.yml：
- API 服務
- PostgreSQL
- Redis
- Nginx（反向代理）

### 練習 5: CI/CD 集成

設置 GitHub Actions 自動構建並推送 Docker 鏡像。

---

下一章：[Chapter 58: 性能優化](58-profiling.md)
