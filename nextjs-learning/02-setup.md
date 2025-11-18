# 第二章：環境設置與第一個 Next.js 專案

## 2.1 環境需求

### 必要軟體

1. **Node.js**（版本 18.17 或更高）
2. **npm** 或 **yarn** 或 **pnpm**（套件管理器）
3. **程式碼編輯器**（推薦 VS Code）

### 檢查環境

```bash
# 檢查 Node.js 版本
node --version
# 應該顯示 v18.17.0 或更高

# 檢查 npm 版本
npm --version

# 檢查 pnpm 版本（可選）
pnpm --version
```

### 安裝 Node.js

**macOS/Linux (使用 nvm)：**
```bash
# 安裝 nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# 安裝 Node.js LTS
nvm install --lts
nvm use --lts
```

**Windows：**
- 從 [nodejs.org](https://nodejs.org) 下載安裝器
- 或使用 [nvm-windows](https://github.com/coreybutler/nvm-windows)

## 2.2 創建第一個 Next.js 專案

### 方法一：使用 create-next-app（推薦）

```bash
# 使用 npx（不需要全域安裝）
npx create-next-app@latest my-nextjs-app

# 或使用 pnpm
pnpm create next-app my-nextjs-app

# 或使用 yarn
yarn create next-app my-nextjs-app
```

### 安裝過程中的選項

安裝時會詢問以下問題：

```
√ Would you like to use TypeScript? ... No / Yes
√ Would you like to use ESLint? ... No / Yes
√ Would you like to use Tailwind CSS? ... No / Yes
√ Would you like to use `src/` directory? ... No / Yes
√ Would you like to use App Router? (recommended) ... No / Yes
√ Would you like to customize the default import alias (@/*)? ... No / Yes
```

**建議選擇（初學者）：**
- TypeScript: **Yes** （強烈建議）
- ESLint: **Yes**
- Tailwind CSS: **No** （先學習基礎）
- `src/` directory: **No** （保持簡單）
- App Router: **No** （先學習 Pages Router）
- Import alias: **No**

### Vue3 對比

如果你熟悉 Vue CLI 或 Vite：

```bash
# Vue3 創建專案
npm create vue@latest    # 使用 create-vue
npm create vite@latest   # 使用 Vite

# Next.js 創建專案
npx create-next-app@latest
```

兩者都提供互動式安裝體驗，但 Next.js 的配置更精簡。

## 2.3 專案初始化

### 進入專案目錄

```bash
cd my-nextjs-app
```

### 啟動開發伺服器

```bash
npm run dev
# 或
pnpm dev
# 或
yarn dev
```

預設會在 `http://localhost:3000` 啟動。

### 開發伺服器輸出

```
   ▲ Next.js 14.0.0
   - Local:        http://localhost:3000
   - Environments: .env

 ✓ Ready in 2.1s
```

## 2.4 專案結構概覽

創建後的專案結構（Pages Router）：

```
my-nextjs-app/
├── pages/              # 頁面目錄（路由）
│   ├── _app.js        # 應用程式根元件
│   ├── _document.js   # HTML 文件結構
│   ├── index.js       # 首頁（/）
│   └── api/           # API 路由
│       └── hello.js   # API 端點範例
├── public/            # 靜態資源
│   ├── favicon.ico
│   └── vercel.svg
├── styles/            # 全域樣式
│   ├── globals.css
│   └── Home.module.css
├── .gitignore
├── package.json
├── next.config.js     # Next.js 配置檔
└── README.md
```

### 與 Vue3 專案對比

| Next.js | Vue3/Nuxt3 | 用途 |
|---------|------------|------|
| `pages/` | `pages/` (Nuxt) | 路由頁面 |
| `public/` | `public/` | 靜態資源 |
| `styles/` | `assets/` | 樣式檔案 |
| `pages/_app.js` | `app.vue` (Nuxt) | 根元件 |
| `next.config.js` | `nuxt.config.ts` | 配置檔 |

## 2.5 理解預設檔案

### pages/index.js（首頁）

```javascript
export default function Home() {
  return (
    <div>
      <h1>歡迎來到 Next.js!</h1>
      <p>開始編輯 pages/index.js</p>
    </div>
  )
}
```

**Vue3 等價元件：**
```vue
<template>
  <div>
    <h1>歡迎來到 Vue3!</h1>
    <p>開始編輯 App.vue</p>
  </div>
</template>
```

### pages/_app.js（全域設定）

```javascript
import '@/styles/globals.css'

export default function App({ Component, pageProps }) {
  return <Component {...pageProps} />
}
```

**作用：**
- 包裹所有頁面元件
- 載入全域樣式
- 添加全域 Layout
- 設定狀態管理（Redux、Context）

**Nuxt3 等價：**
```vue
<!-- app.vue -->
<template>
  <NuxtLayout>
    <NuxtPage />
  </NuxtLayout>
</template>
```

### next.config.js（配置檔）

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
}

module.exports = nextConfig
```

常用配置選項：
```javascript
module.exports = {
  // 嚴格模式（開發時的額外檢查）
  reactStrictMode: true,

  // 圖片域名白名單
  images: {
    domains: ['example.com'],
  },

  // 環境變數
  env: {
    CUSTOM_KEY: 'my-value',
  },

  // 重定向
  async redirects() {
    return [
      {
        source: '/old-path',
        destination: '/new-path',
        permanent: true,
      },
    ]
  },
}
```

## 2.6 第一次修改

### 編輯首頁

修改 `pages/index.js`：

```javascript
export default function Home() {
  return (
    <div style={{ padding: '50px', fontFamily: 'sans-serif' }}>
      <h1 style={{ color: '#0070f3' }}>我的第一個 Next.js 應用</h1>
      <p>這是一個簡單的起始頁面</p>
      <ul>
        <li>快速重新載入（Fast Refresh）</li>
        <li>內建 CSS 支援</li>
        <li>檔案系統路由</li>
      </ul>
    </div>
  )
}
```

儲存後，瀏覽器會自動重新載入（Hot Module Replacement）。

### 添加樣式

創建 `styles/Home.module.css`：

```css
.container {
  padding: 50px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen;
}

.title {
  color: #0070f3;
  font-size: 2.5rem;
  margin-bottom: 1rem;
}

.description {
  font-size: 1.2rem;
  color: #666;
}

.list {
  margin-top: 2rem;
  line-height: 1.8;
}
```

使用 CSS Module：

```javascript
import styles from '@/styles/Home.module.css'

export default function Home() {
  return (
    <div className={styles.container}>
      <h1 className={styles.title}>我的第一個 Next.js 應用</h1>
      <p className={styles.description}>這是一個簡單的起始頁面</p>
      <ul className={styles.list}>
        <li>快速重新載入（Fast Refresh）</li>
        <li>內建 CSS 支援</li>
        <li>檔案系統路由</li>
      </ul>
    </div>
  )
}
```

### CSS 對比

**Vue3（Scoped CSS）：**
```vue
<template>
  <div class="container">
    <h1 class="title">標題</h1>
  </div>
</template>

<style scoped>
.container { padding: 50px; }
.title { color: #0070f3; }
</style>
```

**Next.js（CSS Modules）：**
```javascript
import styles from './Component.module.css'

function Component() {
  return (
    <div className={styles.container}>
      <h1 className={styles.title}>標題</h1>
    </div>
  )
}
```

兩者都實現樣式隔離，但方式不同。

## 2.7 VS Code 推薦擴充

### 必裝擴充

1. **ES7+ React/Redux/React-Native snippets**
   - 提供 React 程式碼片段

2. **Auto Rename Tag**
   - 自動重命名配對的標籤

3. **ESLint**
   - 程式碼品質檢查

4. **Prettier - Code formatter**
   - 程式碼格式化

### 配置 VS Code

創建 `.vscode/settings.json`：

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

## 2.8 常用指令

### package.json scripts

```json
{
  "scripts": {
    "dev": "next dev",           // 開發模式
    "build": "next build",       // 建置生產版本
    "start": "next start",       // 啟動生產伺服器
    "lint": "next lint"          // 程式碼檢查
  }
}
```

### 使用方式

```bash
# 開發模式（Hot Reload）
npm run dev

# 建置生產版本
npm run build

# 啟動生產伺服器（需先 build）
npm run start

# 程式碼檢查
npm run lint
```

## 2.9 除錯技巧

### 使用 console.log

```javascript
export default function Home() {
  console.log('這會在瀏覽器的控制台顯示')

  return <div>首頁</div>
}
```

### 使用 React Developer Tools

1. 安裝 Chrome 擴充：React Developer Tools
2. 開啟開發者工具（F12）
3. 切換到 "Components" 標籤
4. 查看元件樹和 props/state

### VS Code 除錯配置

創建 `.vscode/launch.json`：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: debug server-side",
      "type": "node-terminal",
      "request": "launch",
      "command": "npm run dev"
    },
    {
      "name": "Next.js: debug client-side",
      "type": "chrome",
      "request": "launch",
      "url": "http://localhost:3000"
    }
  ]
}
```

## 2.10 常見問題

### Q1: Port 3000 已被佔用

```bash
# 指定不同的 port
npm run dev -- -p 3001
```

或修改 `package.json`：
```json
{
  "scripts": {
    "dev": "next dev -p 3001"
  }
}
```

### Q2: 模組找不到

```bash
# 清除 node_modules 並重新安裝
rm -rf node_modules
npm install
```

### Q3: 修改後沒有重新載入

- 檢查是否有語法錯誤
- 重啟開發伺服器
- 清除瀏覽器快取

## 2.11 本章小結

- 使用 `create-next-app` 快速建立專案
- 理解基本專案結構
- 學會使用 CSS Modules
- 配置開發環境和編輯器
- 掌握常用開發指令

## 下一章預告

下一章將深入探討 Next.js 的專案結構與檔案組織，幫助你更好地組織程式碼。

## 練習題

1. 創建一個新的 Next.js 專案
2. 修改首頁，添加自己的內容
3. 創建一個 CSS Module 並套用到頁面
4. 嘗試修改 Port 號為 3001
5. 安裝並配置 VS Code 擴充
