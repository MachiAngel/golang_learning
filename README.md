# Go 學習指南 - 為 Node.js 工程師設計

> 從 Node.js 到 Go：一份全面的學習路線圖

## 關於本書

本書專為有 Node.js/JavaScript 背景的開發者設計，幫助你快速掌握 Go 語言。每個章節都會將 Go 的概念與 Node.js 對應的知識點進行對比，讓你能夠快速建立知識遷移。

## 目錄結構

### Part 1: 入門基礎 (Getting Started)
- [第 1 章: 為什麼選擇 Go？Node.js vs Go 對比](./chapters/01-why-go.md)
- [第 2 章: Go 環境安裝與配置](./chapters/02-installation.md)
- [第 3 章: 第一個 Go 程序：Hello World](./chapters/03-hello-world.md)
- [第 4 章: Go 工作空間與模組管理](./chapters/04-modules.md)
- [第 5 章: Go 命令行工具詳解](./chapters/05-go-commands.md)

### Part 2: 基本語法 (Basic Syntax)
- [第 6 章: 變量與常量聲明](./chapters/06-variables-constants.md)
- [第 7 章: 數據類型系統](./chapters/07-data-types.md)
- [第 8 章: 運算符與表達式](./chapters/08-operators.md)
- [第 9 章: 流程控制：if/else](./chapters/09-if-else.md)
- [第 10 章: 循環：for 循環](./chapters/10-loops.md)
- [第 11 章: 函數定義與調用](./chapters/11-functions.md)
- [第 12 章: 多返回值與錯誤處理](./chapters/12-error-handling.md)
- [第 13 章: 指針基礎](./chapters/13-pointers.md)

### Part 3: 複合數據類型 (Composite Types)
- [第 14 章: 數組與切片](./chapters/14-arrays-slices.md)
- [第 15 章: Map 數據結構](./chapters/15-maps.md)
- [第 16 章: 結構體](./chapters/16-structs.md)
- [第 17 章: 方法與接收器](./chapters/17-methods.md)
- [第 18 章: 接口](./chapters/18-interfaces.md)

### Part 4: 高級特性 (Advanced Features)
- [第 19 章: 包與模塊系統](./chapters/19-packages.md)
- [第 20 章: 錯誤處理最佳實踐](./chapters/20-error-best-practices.md)
- [第 21 章: defer、panic 與 recover](./chapters/21-defer-panic-recover.md)
- [第 22 章: Goroutines：Go 的並發模型](./chapters/22-goroutines.md)
- [第 23 章: Channels：goroutine 間通信](./chapters/23-channels.md)
- [第 24 章: Select 語句：多路復用](./chapters/24-select.md)
- [第 25 章: 互斥鎖與同步](./chapters/25-mutex-sync.md)
- [第 26 章: Context 包：請求生命週期管理](./chapters/26-context.md)

### Part 5: 標準庫 (Standard Library)
- [第 27 章: fmt 包：格式化輸入輸出](./chapters/27-fmt.md)
- [第 28 章: strings 與 strconv：字符串處理](./chapters/28-strings.md)
- [第 29 章: time 包：時間處理](./chapters/29-time.md)
- [第 30 章: json 包：JSON 處理](./chapters/30-json.md)
- [第 31 章: http 包：HTTP 客戶端](./chapters/31-http-client.md)
- [第 32 章: io 與文件操作](./chapters/32-io-files.md)
- [第 33 章: os 包：操作系統交互](./chapters/33-os.md)

### Part 6: Web 開發 (Web Development)
- [第 34 章: net/http：構建 HTTP 服務器](./chapters/34-http-server.md)
- [第 35 章: 路由與中間件模式](./chapters/35-routing-middleware.md)
- [第 36 章: Gin 框架入門](./chapters/36-gin-framework.md)
- [第 37 章: Echo 框架介紹](./chapters/37-echo-framework.md)
- [第 38 章: Fiber 框架介紹](./chapters/38-fiber-framework.md)
- [第 39 章: 框架對比：Gin vs Echo vs Fiber](./chapters/39-framework-comparison.md)
- [第 40 章: RESTful API 實戰](./chapters/40-restful-api.md)

### Part 7: 數據庫操作 (Database)
- [第 41 章: database/sql 標準接口](./chapters/41-database-sql.md)
- [第 42 章: GORM：Go 的 ORM](./chapters/42-gorm.md)
- [第 43 章: PostgreSQL 集成實戰](./chapters/43-postgresql.md)
- [第 44 章: MongoDB 與 Go](./chapters/44-mongodb.md)
- [第 45 章: Redis 集成](./chapters/45-redis.md)

### Part 8: 測試與工具 (Testing & Tools)
- [第 46 章: testing 包：單元測試](./chapters/46-unit-testing.md)
- [第 47 章: 基準測試與性能優化](./chapters/47-benchmarking.md)
- [第 48 章: 表格驅動測試](./chapters/48-table-driven-tests.md)
- [第 49 章: Mock 與測試替身](./chapters/49-mocking.md)

### Part 9: 實戰項目 (Real-world Projects)
- [第 50 章: 項目結構最佳實踐](./chapters/50-project-structure.md)
- [第 51 章: 構建 RESTful API 完整項目](./chapters/51-api-project.md)
- [第 52 章: 微服務架構入門](./chapters/52-microservices.md)
- [第 53 章: gRPC 與 Protocol Buffers](./chapters/53-grpc.md)
- [第 54 章: WebSocket 實時通信](./chapters/54-websocket.md)
- [第 55 章: JWT 認證實現](./chapters/55-jwt-auth.md)

### Part 10: 部署與運維 (Deployment & DevOps)
- [第 56 章: 編譯與交叉編譯](./chapters/56-compilation.md)
- [第 57 章: Docker 容器化](./chapters/57-docker.md)
- [第 58 章: 性能優化與 profiling](./chapters/58-profiling.md)
- [第 59 章: 生產環境最佳實踐](./chapters/59-production.md)
- [第 60 章: 從 Node.js 遷移到 Go 的策略](./chapters/60-migration-strategy.md)

### Part 11: 2025 現代 Go 開發 (Modern Go 2025)
- [第 61 章: 2025 年 Go 寫作風格演進與最佳實踐](./chapters/61-golang-evolution-2025.md)
- [第 62 章: 2025 年實際 Web 專案範例](./chapters/62-web-project-2025.md)

## 如何使用本書

1. **循序漸進**：建議按順序閱讀，每個章節都建立在前面的基礎上
2. **動手實踐**：每章都包含可執行的代碼示例，請務必親自嘗試
3. **對比學習**：注意每章中 Node.js 與 Go 的對比說明
4. **項目實戰**：完成 Part 9 的實戰項目，鞏固所學知識

## 前置要求

- 熟悉 JavaScript/Node.js 開發
- 了解基本的命令行操作
- 有 Web 開發經驗更佳

## 學習路線建議

### 快速上手 (1-2 週)
- Part 1-2: 基礎語法和環境配置
- Part 3: 複合數據類型

### 進階學習 (2-3 週)
- Part 4: 掌握 Go 的並發特性
- Part 5-6: 標準庫和 Web 開發

### 實戰應用 (3-4 週)
- Part 7-8: 數據庫和測試
- Part 9-10: 完整項目和部署

## 貢獻與反饋

歡迎提出問題和建議，幫助改進本書內容。

---

**開始你的 Go 學習之旅吧！** 🚀
