# Next.js 學習指南

> 專為有 Vue 3 經驗的開發者設計的完整 Next.js 學習路徑

## 📖 課程簡介

本學習指南包含 23 個章節，從基礎到進階，涵蓋 Next.js 框架的所有核心概念、部署策略、以及實際應用整合。每個章節都提供豐富的程式碼範例和實戰案例。

## 🎯 適合對象

- 有 Vue 3 / Nuxt.js 開發經驗的工程師
- 想要學習 React 生態系的前端開發者
- 需要構建 SEO 友好網站的開發團隊
- 對 SSR/SSG 技術感興趣的開發者

## 📚 課程大綱

### 第一部分：基礎入門 (1-8 章)

| 章節 | 標題 | 主要內容 |
|------|------|----------|
| [01](./01-introduction.md) | Next.js 簡介與 Vue3 對比 | 框架特色、概念對比、選擇建議 |
| [02](./02-setup.md) | 環境設置與第一個 Next.js 專案 | 環境安裝、專案創建、開發工具配置 |
| [03](./03-project-structure.md) | 專案結構與檔案組織 | 目錄結構、特殊檔案、最佳實踐 |
| [04](./04-pages-vs-app-router.md) | Pages Router vs App Router | 兩種路由模式對比、選擇指南 |
| [05](./05-routing-basics.md) | 路由系統基礎 | 檔案路由、導航、useRouter Hook |
| [06](./06-dynamic-routes.md) | 動態路由與參數處理 | 動態路由、Catch-all、參數處理 |
| [07](./07-getStaticProps.md) | 資料獲取：getStaticProps | SSG 原理、ISR、實戰範例 |
| [08](./08-getServerSideProps.md) | 資料獲取：getServerSideProps | SSR 原理、Context 參數、認證實作 |

### 第二部分：進階技術 (9-16 章)

| 章節 | 標題 | 主要內容 |
|------|------|----------|
| [09](./09-getStaticPaths.md) | 資料獲取：getStaticPaths 與動態路由 | 動態靜態生成、Fallback 模式 |
| [10](./10-ssg.md) | SSG (Static Site Generation) 深入解析 | 靜態生成策略、CMS 整合、優化技巧 |
| [11](./11-ssr.md) | SSR (Server-Side Rendering) 深入解析 | 伺服器渲染、個人化內容、效能優化 |
| [12](./12-isr.md) | ISR (Incremental Static Regeneration) | 漸進式靜態生成、On-Demand Revalidation |
| [13](./13-api-routes.md) | API Routes - 建立後端 API | REST API、CRUD、中介軟體、認證 |
| [14](./14-css-styling.md) | CSS 與樣式處理 | CSS Modules、Tailwind CSS、設計系統 |
| [15](./15-image-optimization.md) | 圖片優化與 next/image 組件 | 圖片優化、響應式圖片、效能提升 |
| [16](./16-seo.md) | SEO 優化與 Meta Tags 管理 | Meta Tags、Open Graph、結構化資料 |

### 第三部分：部署實戰 (17-20 章)

| 章節 | 標題 | 主要內容 |
|------|------|----------|
| [17](./17-deploy-vercel.md) | 部署到 Vercel | 官方推薦平台、CI/CD、環境變數 |
| [18](./18-deploy-gcp.md) | 部署到 Google Cloud Platform | Cloud Run、App Engine、容器化 |
| [19](./19-deploy-cloudflare.md) | 部署到 Cloudflare Pages | Edge Runtime、Workers、免費額度 |
| [20](./20-deploy-heroku.md) | 部署到 Heroku | Dyno 配置、資料庫整合、Pipeline |

### 第四部分：整合與優化 (21-23 章)

| 章節 | 標題 | 主要內容 |
|------|------|----------|
| [21](./21-google-ads.md) | Google AdSense 與 Google Ads 整合 | AdSense 整合、Script 優化、廣告策略 |
| [22](./22-performance.md) | 效能優化最佳實踐 | Core Web Vitals、程式碼分割、快取策略 |
| [23](./23-troubleshooting.md) | 常見問題與除錯技巧 | 錯誤排查、效能監控、遷移陷阱 |

## 🚀 快速開始

### 1. 前置需求

- Node.js 18.17 或更高版本
- npm、yarn 或 pnpm 套件管理器
- 基本的 React 知識（有 Vue 3 經驗會更容易理解）

### 2. 建立第一個專案

```bash
# 使用 create-next-app 建立專案
npx create-next-app@latest my-nextjs-app

# 進入專案目錄
cd my-nextjs-app

# 啟動開發伺服器
npm run dev
```

### 3. 學習路徑建議

**初學者路徑** (2-3 週)
```
第 1-8 章 → 建立一個簡單的部落格 → 第 17 章 (部署到 Vercel)
```

**進階路徑** (4-6 週)
```
第 1-16 章 → 建立完整的電商網站 → 第 17-20 章 (多平台部署) → 第 21-23 章 (優化整合)
```

**從 Vue 3 遷移路徑** (1-2 週)
```
第 1 章 (對比) → 第 4-8 章 (路由與資料) → 第 13 章 (API) → 第 22-23 章 (優化除錯)
```

## 💡 核心特色

### ✨ 針對 Vue 3 開發者設計

每個章節都包含與 Vue 3 / Nuxt.js 的對比，幫助你快速理解：

- **元件系統**：從 `.vue` 單檔案到 `.jsx/.tsx`
- **響應式資料**：從 `ref/reactive` 到 `useState/useReducer`
- **生命週期**：從 Vue 生命週期到 React Hooks
- **路由系統**：從 Vue Router 到 Next.js 檔案路由

### 📝 豐富的實戰範例

- 部落格系統（SSG + ISR）
- 電商平台（SSR + API Routes）
- 新聞網站（ISR + SEO 優化）
- 儀表板（API Routes + 認證）

### 🎨 完整的程式碼示範

所有章節都包含完整、可運行的程式碼範例：

- TypeScript 類型定義
- 錯誤處理
- 載入狀態
- 最佳實踐

## 📊 框架特色對比

### Next.js vs Vue 3 / Nuxt 3

| 特性 | Next.js | Vue 3 / Nuxt 3 |
|------|---------|----------------|
| **框架類型** | React 元框架 | Vue 元框架 |
| **學習曲線** | 中等（需要 React 基礎） | 較平緩 |
| **渲染模式** | SSG、SSR、ISR、CSR | SSG、SSR、Hybrid |
| **路由系統** | 檔案系統路由 | 檔案系統路由 |
| **資料獲取** | getStaticProps、getServerSideProps | useFetch、useAsyncData |
| **生態系統** | 非常成熟（React） | 快速成長（Vue） |
| **部署選擇** | Vercel (官方)、多平台支援 | Netlify、Vercel、多平台 |
| **開發體驗** | 優秀 | 非常優秀 |
| **企業採用** | 極高 | 高且增長中 |

## 🛠️ 推薦工具

### 開發工具

- **編輯器**：VS Code、WebStorm
- **擴充套件**：ES7+ React/Redux snippets、Tailwind CSS IntelliSense
- **除錯**：React Developer Tools、Next.js DevTools

### 部署平台

- **Vercel** ⭐ 推薦：官方平台，零配置
- **Cloudflare Pages**：Edge 運算，免費額度最高
- **Google Cloud Platform**：企業級，可擴展
- **Heroku**：傳統 PaaS，簡單易用

### 整合服務

- **CMS**：Contentful、Sanity、Strapi
- **資料庫**：PostgreSQL、MongoDB、Prisma ORM
- **認證**：NextAuth.js、Auth0、Clerk
- **分析**：Google Analytics、Vercel Analytics

## 📖 學習資源

### 官方資源

- [Next.js 官方文件](https://nextjs.org/docs)
- [Next.js 學習課程](https://nextjs.org/learn)
- [Next.js GitHub](https://github.com/vercel/next.js)

### 社群資源

- [Next.js Discord](https://discord.gg/nextjs)
- [Next.js 繁體中文文件](https://nextjs.tw/)
- [Awesome Next.js](https://github.com/unicodeveloper/awesome-nextjs)

### 進階學習

- [React 官方文件](https://react.dev/)
- [TypeScript 手冊](https://www.typescriptlang.org/docs/)
- [Vercel 部落格](https://vercel.com/blog)

## 🎓 實戰專案建議

### 初級專案

1. **個人部落格** (第 1-8 章)
   - 使用 SSG 生成文章頁面
   - Markdown 文件管理
   - 基本 SEO 優化

2. **產品展示網站** (第 1-10 章)
   - 靜態產品頁面
   - 圖片優化
   - 響應式設計

### 中級專案

3. **新聞網站** (第 1-16 章)
   - ISR 自動更新內容
   - CMS 整合
   - 完整 SEO 設定

4. **電商平台** (第 1-16 章)
   - 產品列表與詳情頁
   - 購物車功能
   - API Routes 後端

### 高級專案

5. **SaaS 應用程式** (所有章節)
   - 用戶認證與授權
   - 個人化儀表板
   - 訂閱與付款整合
   - 多平台部署

6. **內容平台** (所有章節)
   - 多角色權限系統
   - 即時資料更新
   - Google Ads 變現
   - 效能監控

## 📝 練習建議

每個章節都包含練習題，建議：

1. **閱讀理論**：仔細閱讀章節內容和範例
2. **動手實作**：完成章節末尾的練習題
3. **擴展應用**：嘗試將概念應用到自己的專案
4. **查閱文件**：遇到問題時查閱官方文件
5. **社群討論**：加入 Discord 社群討論問題

## 🤝 貢獻

歡迎提出建議和改進：

- 發現錯誤或錯別字
- 建議新的章節主題
- 分享你的學習心得
- 提供更好的範例程式碼

## 📄 授權

本學習指南採用 MIT 授權。

## 🙏 致謝

感謝 Next.js 團隊和 Vercel 提供如此優秀的框架和平台。

---

## 💬 常見問題

### Q: 我需要先學 React 嗎？

A: 基礎的 React 知識會有幫助，但如果你有 Vue 3 經驗，可以邊學邊補充 React 概念。第 1 章有詳細的對比說明。

### Q: Next.js 和 Nuxt.js 該選哪個？

A: 兩者都很優秀。Next.js 生態系更成熟，企業採用率更高；Nuxt.js 對 Vue 開發者更友善。建議根據團隊技術棧選擇。

### Q: 學完需要多久時間？

A:
- **快速入門**：1-2 週（第 1-8 章）
- **完整學習**：4-6 週（所有章節）
- **精通應用**：3-6 個月（實戰專案）

### Q: 部署到哪個平台最好？

A:
- **初學者**：Vercel（零配置，免費額度）
- **預算有限**：Cloudflare Pages（免費額度最高）
- **企業應用**：Google Cloud Platform（可擴展性）
- **傳統應用**：Heroku（簡單易用）

### Q: 如何整合現有的 Vue 3 專案？

A: Next.js 使用 React，無法直接整合 Vue 3 元件。建議：
1. 評估是否真的需要遷移
2. 如需遷移，採用漸進式重寫
3. 或考慮使用 Nuxt.js（Vue 版本的 Next.js）

---

**開始你的 Next.js 學習之旅吧！** 🚀

從 [第 1 章：Next.js 簡介與 Vue3 對比](./01-introduction.md) 開始！
