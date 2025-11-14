# Chapter 52: 微服務架構

## 概述

微服務架構將大型應用拆分為多個小型、獨立的服務，每個服務專注於特定的業務功能。本章將介紹如何使用 Go 構建微服務，包括服務拆分、服務間通信、服務發現等核心概念。

## Node.js vs Go 微服務對比

### Node.js 微服務生態
- **框架**: Seneca, Moleculer, NestJS
- **通信**: HTTP/REST, Message Queue (RabbitMQ, Kafka)
- **優勢**: 快速開發、豐富的 npm 生態
- **挑戰**: 單線程限制、內存管理

### Go 微服務生態
- **框架**: Go Kit, Micro, Gin
- **通信**: HTTP/REST, gRPC, Message Queue
- **優勢**: 高性能、並發處理、靜態類型
- **挑戰**: 相對較新的生態系統

## 微服務架構設計

### 示例系統架構

我們將構建一個電商系統的微服務架構：

```
電商系統
├── API Gateway          (統一入口)
├── User Service        (用戶服務)
├── Product Service     (商品服務)
├── Order Service       (訂單服務)
└── Notification Service (通知服務)
```

### 服務拆分原則

1. **單一職責**: 每個服務只負責一個業務領域
2. **獨立部署**: 服務可以獨立部署和擴展
3. **數據隔離**: 每個服務有自己的數據庫
4. **鬆耦合**: 服務間通過 API 通信

## 實戰項目：構建微服務系統

### 項目結構

```
microservices/
├── api-gateway/           # API 網關
│   ├── cmd/
│   │   └── main.go
│   └── internal/
│       └── proxy/
├── user-service/          # 用戶服務
│   ├── cmd/
│   │   └── main.go
│   └── internal/
│       ├── handlers/
│       ├── models/
│       └── repository/
├── product-service/       # 商品服務
│   └── ...
├── order-service/         # 訂單服務
│   └── ...
├── shared/                # 共享代碼
│   ├── config/
│   ├── logger/
│   └── middleware/
└── docker-compose.yml
```

## 1. 用戶服務 (User Service)

### user-service/internal/models/user.go
```go
package models

import "time"

type User struct {
    ID        int       `json:"id" db:"id"`
    Email     string    `json:"email" db:"email"`
    Name      string    `json:"name" db:"name"`
    CreatedAt time.Time `json:"created_at" db:"created_at"`
}

type CreateUserRequest struct {
    Email string `json:"email" validate:"required,email"`
    Name  string `json:"name" validate:"required"`
}
```

### user-service/internal/handlers/user.go
```go
package handlers

import (
    "encoding/json"
    "net/http"
    "strconv"
    "user-service/internal/models"
    "user-service/internal/repository"

    "github.com/gorilla/mux"
)

type UserHandler struct {
    repo *repository.UserRepository
}

func NewUserHandler(repo *repository.UserRepository) *UserHandler {
    return &UserHandler{repo: repo}
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "Invalid user ID", http.StatusBadRequest)
        return
    }

    user, err := h.repo.GetByID(id)
    if err != nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req models.CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    user := &models.User{
        Email: req.Email,
        Name:  req.Name,
    }

    if err := h.repo.Create(user); err != nil {
        http.Error(w, "Failed to create user", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func (h *UserHandler) ListUsers(w http.ResponseWriter, r *http.Request) {
    users, err := h.repo.List()
    if err != nil {
        http.Error(w, "Failed to fetch users", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}
```

### user-service/cmd/main.go
```go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "os"
    "user-service/internal/handlers"
    "user-service/internal/repository"

    "github.com/gorilla/mux"
    _ "github.com/lib/pq"
)

func main() {
    // 連接數據庫
    dbURL := os.Getenv("DATABASE_URL")
    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // 初始化
    repo := repository.NewUserRepository(db)
    handler := handlers.NewUserHandler(repo)

    // 設置路由
    r := mux.NewRouter()
    r.HandleFunc("/users", handler.ListUsers).Methods("GET")
    r.HandleFunc("/users", handler.CreateUser).Methods("POST")
    r.HandleFunc("/users/{id}", handler.GetUser).Methods("GET")

    // 健康檢查
    r.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    }).Methods("GET")

    // 啟動服務
    port := os.Getenv("PORT")
    if port == "" {
        port = "8081"
    }

    log.Printf("User service starting on port %s", port)
    if err := http.ListenAndServe(":"+port, r); err != nil {
        log.Fatal(err)
    }
}
```

## 2. 商品服務 (Product Service)

### product-service/internal/models/product.go
```go
package models

import "time"

type Product struct {
    ID          int       `json:"id" db:"id"`
    Name        string    `json:"name" db:"name"`
    Description string    `json:"description" db:"description"`
    Price       float64   `json:"price" db:"price"`
    Stock       int       `json:"stock" db:"stock"`
    CreatedAt   time.Time `json:"created_at" db:"created_at"`
}

type CreateProductRequest struct {
    Name        string  `json:"name" validate:"required"`
    Description string  `json:"description"`
    Price       float64 `json:"price" validate:"required,gt=0"`
    Stock       int     `json:"stock" validate:"required,gte=0"`
}
```

### product-service/cmd/main.go
```go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "os"
    "product-service/internal/handlers"
    "product-service/internal/repository"

    "github.com/gorilla/mux"
    _ "github.com/lib/pq"
)

func main() {
    dbURL := os.Getenv("DATABASE_URL")
    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    repo := repository.NewProductRepository(db)
    handler := handlers.NewProductHandler(repo)

    r := mux.NewRouter()
    r.HandleFunc("/products", handler.ListProducts).Methods("GET")
    r.HandleFunc("/products", handler.CreateProduct).Methods("POST")
    r.HandleFunc("/products/{id}", handler.GetProduct).Methods("GET")
    r.HandleFunc("/products/{id}/stock", handler.UpdateStock).Methods("PATCH")

    r.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    }).Methods("GET")

    port := os.Getenv("PORT")
    if port == "" {
        port = "8082"
    }

    log.Printf("Product service starting on port %s", port)
    if err := http.ListenAndServe(":"+port, r); err != nil {
        log.Fatal(err)
    }
}
```

## 3. API Gateway

API Gateway 作為統一入口，負責路由請求到相應的微服務。

### api-gateway/internal/proxy/proxy.go
```go
package proxy

import (
    "fmt"
    "io"
    "net/http"
    "net/url"
)

type ServiceRegistry struct {
    services map[string]string
}

func NewServiceRegistry() *ServiceRegistry {
    return &ServiceRegistry{
        services: map[string]string{
            "user":    "http://user-service:8081",
            "product": "http://product-service:8082",
            "order":   "http://order-service:8083",
        },
    }
}

func (sr *ServiceRegistry) GetServiceURL(service string) (string, error) {
    url, ok := sr.services[service]
    if !ok {
        return "", fmt.Errorf("service not found: %s", service)
    }
    return url, nil
}

type Gateway struct {
    registry *ServiceRegistry
}

func NewGateway(registry *ServiceRegistry) *Gateway {
    return &Gateway{registry: registry}
}

func (g *Gateway) ProxyRequest(service string) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 獲取服務 URL
        serviceURL, err := g.registry.GetServiceURL(service)
        if err != nil {
            http.Error(w, err.Error(), http.StatusBadGateway)
            return
        }

        // 構建目標 URL
        targetURL, err := url.Parse(serviceURL + r.URL.Path)
        if err != nil {
            http.Error(w, "Invalid URL", http.StatusInternalServerError)
            return
        }
        targetURL.RawQuery = r.URL.RawQuery

        // 創建新請求
        proxyReq, err := http.NewRequest(r.Method, targetURL.String(), r.Body)
        if err != nil {
            http.Error(w, "Failed to create request", http.StatusInternalServerError)
            return
        }

        // 複製請求頭
        for key, values := range r.Header {
            for _, value := range values {
                proxyReq.Header.Add(key, value)
            }
        }

        // 發送請求
        client := &http.Client{}
        resp, err := client.Do(proxyReq)
        if err != nil {
            http.Error(w, "Service unavailable", http.StatusServiceUnavailable)
            return
        }
        defer resp.Body.Close()

        // 複製響應頭
        for key, values := range resp.Header {
            for _, value := range values {
                w.Header().Add(key, value)
            }
        }

        // 設置狀態碼和響應體
        w.WriteHeader(resp.StatusCode)
        io.Copy(w, resp.Body)
    }
}
```

### api-gateway/cmd/main.go
```go
package main

import (
    "log"
    "net/http"
    "os"
    "api-gateway/internal/proxy"

    "github.com/gorilla/mux"
)

func main() {
    registry := proxy.NewServiceRegistry()
    gateway := proxy.NewGateway(registry)

    r := mux.NewRouter()

    // 用戶服務路由
    r.PathPrefix("/api/users").HandlerFunc(gateway.ProxyRequest("user"))

    // 商品服務路由
    r.PathPrefix("/api/products").HandlerFunc(gateway.ProxyRequest("product"))

    // 訂單服務路由
    r.PathPrefix("/api/orders").HandlerFunc(gateway.ProxyRequest("order"))

    // 健康檢查
    r.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("Gateway OK"))
    }).Methods("GET")

    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    log.Printf("API Gateway starting on port %s", port)
    if err := http.ListenAndServe(":"+port, r); err != nil {
        log.Fatal(err)
    }
}
```

## 4. 訂單服務 (Order Service)

訂單服務需要與其他服務通信，演示服務間調用。

### order-service/internal/models/order.go
```go
package models

import "time"

type Order struct {
    ID         int       `json:"id" db:"id"`
    UserID     int       `json:"user_id" db:"user_id"`
    ProductID  int       `json:"product_id" db:"product_id"`
    Quantity   int       `json:"quantity" db:"quantity"`
    TotalPrice float64   `json:"total_price" db:"total_price"`
    Status     string    `json:"status" db:"status"` // pending, confirmed, cancelled
    CreatedAt  time.Time `json:"created_at" db:"created_at"`
}

type CreateOrderRequest struct {
    UserID    int `json:"user_id" validate:"required"`
    ProductID int `json:"product_id" validate:"required"`
    Quantity  int `json:"quantity" validate:"required,gt=0"`
}
```

### order-service/internal/client/client.go
```go
package client

import (
    "encoding/json"
    "fmt"
    "net/http"
)

// UserClient 用戶服務客戶端
type UserClient struct {
    baseURL string
    client  *http.Client
}

func NewUserClient(baseURL string) *UserClient {
    return &UserClient{
        baseURL: baseURL,
        client:  &http.Client{},
    }
}

type User struct {
    ID    int    `json:"id"`
    Email string `json:"email"`
    Name  string `json:"name"`
}

func (c *UserClient) GetUser(id int) (*User, error) {
    url := fmt.Sprintf("%s/users/%d", c.baseURL, id)

    resp, err := c.client.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("user service returned status %d", resp.StatusCode)
    }

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        return nil, err
    }

    return &user, nil
}

// ProductClient 商品服務客戶端
type ProductClient struct {
    baseURL string
    client  *http.Client
}

func NewProductClient(baseURL string) *ProductClient {
    return &ProductClient{
        baseURL: baseURL,
        client:  &http.Client{},
    }
}

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
    Stock int     `json:"stock"`
}

func (c *ProductClient) GetProduct(id int) (*Product, error) {
    url := fmt.Sprintf("%s/products/%d", c.baseURL, id)

    resp, err := c.client.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("product service returned status %d", resp.StatusCode)
    }

    var product Product
    if err := json.NewDecoder(resp.Body).Decode(&product); err != nil {
        return nil, err
    }

    return &product, nil
}
```

### order-service/internal/service/order.go
```go
package service

import (
    "fmt"
    "order-service/internal/client"
    "order-service/internal/models"
    "order-service/internal/repository"
)

type OrderService struct {
    repo          *repository.OrderRepository
    userClient    *client.UserClient
    productClient *client.ProductClient
}

func NewOrderService(
    repo *repository.OrderRepository,
    userClient *client.UserClient,
    productClient *client.ProductClient,
) *OrderService {
    return &OrderService{
        repo:          repo,
        userClient:    userClient,
        productClient: productClient,
    }
}

func (s *OrderService) CreateOrder(req *models.CreateOrderRequest) (*models.Order, error) {
    // 1. 驗證用戶存在
    user, err := s.userClient.GetUser(req.UserID)
    if err != nil {
        return nil, fmt.Errorf("invalid user: %w", err)
    }

    // 2. 驗證商品存在和庫存
    product, err := s.productClient.GetProduct(req.ProductID)
    if err != nil {
        return nil, fmt.Errorf("invalid product: %w", err)
    }

    if product.Stock < req.Quantity {
        return nil, fmt.Errorf("insufficient stock")
    }

    // 3. 計算總價
    totalPrice := product.Price * float64(req.Quantity)

    // 4. 創建訂單
    order := &models.Order{
        UserID:     user.ID,
        ProductID:  product.ID,
        Quantity:   req.Quantity,
        TotalPrice: totalPrice,
        Status:     "pending",
    }

    if err := s.repo.Create(order); err != nil {
        return nil, err
    }

    // 5. TODO: 更新商品庫存（應該通過消息隊列或事務處理）
    // 這裡簡化處理

    return order, nil
}
```

### order-service/cmd/main.go
```go
package main

import (
    "database/sql"
    "log"
    "net/http"
    "os"
    "order-service/internal/client"
    "order-service/internal/handlers"
    "order-service/internal/repository"
    "order-service/internal/service"

    "github.com/gorilla/mux"
    _ "github.com/lib/pq"
)

func main() {
    // 數據庫連接
    dbURL := os.Getenv("DATABASE_URL")
    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // 初始化客戶端
    userServiceURL := os.Getenv("USER_SERVICE_URL")
    productServiceURL := os.Getenv("PRODUCT_SERVICE_URL")

    userClient := client.NewUserClient(userServiceURL)
    productClient := client.NewProductClient(productServiceURL)

    // 初始化服務
    repo := repository.NewOrderRepository(db)
    orderService := service.NewOrderService(repo, userClient, productClient)
    handler := handlers.NewOrderHandler(orderService)

    // 設置路由
    r := mux.NewRouter()
    r.HandleFunc("/orders", handler.CreateOrder).Methods("POST")
    r.HandleFunc("/orders/{id}", handler.GetOrder).Methods("GET")
    r.HandleFunc("/orders", handler.ListOrders).Methods("GET")

    r.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    }).Methods("GET")

    port := os.Getenv("PORT")
    if port == "" {
        port = "8083"
    }

    log.Printf("Order service starting on port %s", port)
    if err := http.ListenAndServe(":"+port, r); err != nil {
        log.Fatal(err)
    }
}
```

## 5. Docker Compose 配置

### docker-compose.yml
```yaml
version: '3.8'

services:
  # PostgreSQL 數據庫
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  # 用戶服務
  user-service:
    build: ./user-service
    ports:
      - "8081:8081"
    environment:
      PORT: 8081
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/userdb?sslmode=disable
    depends_on:
      - postgres

  # 商品服務
  product-service:
    build: ./product-service
    ports:
      - "8082:8082"
    environment:
      PORT: 8082
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/productdb?sslmode=disable
    depends_on:
      - postgres

  # 訂單服務
  order-service:
    build: ./order-service
    ports:
      - "8083:8083"
    environment:
      PORT: 8083
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/orderdb?sslmode=disable
      USER_SERVICE_URL: http://user-service:8081
      PRODUCT_SERVICE_URL: http://product-service:8082
    depends_on:
      - postgres
      - user-service
      - product-service

  # API Gateway
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      PORT: 8080
    depends_on:
      - user-service
      - product-service
      - order-service

volumes:
  postgres-data:
```

## 6. 服務發現（使用 Consul）

### 集成 Consul 進行服務註冊和發現

```go
package discovery

import (
    "fmt"
    "github.com/hashicorp/consul/api"
)

type ServiceDiscovery struct {
    client *api.Client
}

func NewServiceDiscovery(consulAddr string) (*ServiceDiscovery, error) {
    config := api.DefaultConfig()
    config.Address = consulAddr

    client, err := api.NewClient(config)
    if err != nil {
        return nil, err
    }

    return &ServiceDiscovery{client: client}, nil
}

// RegisterService 註冊服務
func (sd *ServiceDiscovery) RegisterService(name, address string, port int) error {
    registration := &api.AgentServiceRegistration{
        ID:      fmt.Sprintf("%s-%s-%d", name, address, port),
        Name:    name,
        Address: address,
        Port:    port,
        Check: &api.AgentServiceCheck{
            HTTP:     fmt.Sprintf("http://%s:%d/health", address, port),
            Interval: "10s",
            Timeout:  "3s",
        },
    }

    return sd.client.Agent().ServiceRegister(registration)
}

// DiscoverService 發現服務
func (sd *ServiceDiscovery) DiscoverService(name string) (string, error) {
    services, _, err := sd.client.Health().Service(name, "", true, nil)
    if err != nil {
        return "", err
    }

    if len(services) == 0 {
        return "", fmt.Errorf("no healthy instances of %s found", name)
    }

    // 簡單的負載均衡：返回第一個健康的實例
    service := services[0]
    address := fmt.Sprintf("http://%s:%d", service.Service.Address, service.Service.Port)

    return address, nil
}
```

## Node.js 對應版本

### API Gateway (Node.js + Express)

```javascript
// api-gateway/index.js
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');

const app = express();

// 用戶服務代理
app.use('/api/users', createProxyMiddleware({
    target: 'http://user-service:3001',
    changeOrigin: true,
    pathRewrite: {
        '^/api/users': '/users'
    }
}));

// 商品服務代理
app.use('/api/products', createProxyMiddleware({
    target: 'http://product-service:3002',
    changeOrigin: true,
    pathRewrite: {
        '^/api/products': '/products'
    }
}));

// 訂單服務代理
app.use('/api/orders', createProxyMiddleware({
    target: 'http://order-service:3003',
    changeOrigin: true,
    pathRewrite: {
        '^/api/orders': '/orders'
    }
}));

app.get('/health', (req, res) => {
    res.send('Gateway OK');
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`API Gateway running on port ${PORT}`);
});
```

### 用戶服務 (Node.js)

```javascript
// user-service/index.js
const express = require('express');
const userRoutes = require('./routes/user');

const app = express();
app.use(express.json());

app.use('/users', userRoutes);

app.get('/health', (req, res) => {
    res.send('OK');
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
    console.log(`User service running on port ${PORT}`);
});
```

### 訂單服務調用其他服務 (Node.js)

```javascript
// order-service/services/orderService.js
const axios = require('axios');

class OrderService {
    constructor() {
        this.userServiceUrl = process.env.USER_SERVICE_URL;
        this.productServiceUrl = process.env.PRODUCT_SERVICE_URL;
    }

    async createOrder(userId, productId, quantity) {
        // 驗證用戶
        const user = await this.getUser(userId);
        if (!user) {
            throw new Error('User not found');
        }

        // 驗證商品和庫存
        const product = await this.getProduct(productId);
        if (!product) {
            throw new Error('Product not found');
        }

        if (product.stock < quantity) {
            throw new Error('Insufficient stock');
        }

        // 創建訂單
        const totalPrice = product.price * quantity;
        const order = {
            userId,
            productId,
            quantity,
            totalPrice,
            status: 'pending'
        };

        // 保存到數據庫...
        return order;
    }

    async getUser(userId) {
        try {
            const response = await axios.get(
                `${this.userServiceUrl}/users/${userId}`
            );
            return response.data;
        } catch (error) {
            return null;
        }
    }

    async getProduct(productId) {
        try {
            const response = await axios.get(
                `${this.productServiceUrl}/products/${productId}`
            );
            return response.data;
        } catch (error) {
            return null;
        }
    }
}

module.exports = new OrderService();
```

## 重點總結

### 微服務架構核心概念

1. **服務拆分**
   - 按業務領域劃分
   - 每個服務獨立部署
   - 數據庫隔離

2. **服務通信**
   - HTTP/REST（簡單，但可能有性能問題）
   - gRPC（高性能，下一章介紹）
   - 消息隊列（異步通信）

3. **API Gateway**
   - 統一入口
   - 請求路由
   - 認證授權
   - 限流熔斷

4. **服務發現**
   - Consul
   - Etcd
   - Kubernetes DNS

### Go vs Node.js 微服務對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 性能 | 中等 | 高 |
| 並發處理 | 事件循環 | Goroutine |
| 內存使用 | 較高 | 較低 |
| 啟動時間 | 快 | 非常快 |
| 部署大小 | 需要 Node.js 運行時 | 單個二進制文件 |
| 生態系統 | 豐富 | 成長中 |

### 最佳實踐

1. **服務隔離**
   - 每個服務獨立數據庫
   - 避免共享狀態

2. **容錯設計**
   - 熔斷器（Circuit Breaker）
   - 重試機制
   - 超時控制

3. **監控和日誌**
   - 集中式日誌收集
   - 分布式追蹤（Jaeger, Zipkin）
   - 健康檢查

4. **部署策略**
   - 容器化（Docker）
   - 編排（Kubernetes）
   - CI/CD

## 練習題

### 練習 1: 實現熔斷器

為服務間調用添加熔斷器模式，防止級聯失敗。

```go
type CircuitBreaker struct {
    maxFailures int
    timeout     time.Duration
    failures    int
    lastFailure time.Time
    state       string // closed, open, half-open
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    // 實現熔斷邏輯
}
```

### 練習 2: 實現重試機制

為 HTTP 客戶端添加指數退避的重試機制。

### 練習 3: 添加分布式追蹤

集成 OpenTelemetry 實現請求追蹤。

### 練習 4: 實現 Saga 模式

為訂單創建實現分布式事務（Saga 模式）。

### 練習 5: 消息隊列集成

使用 RabbitMQ 或 Kafka 實現服務間異步通信。

提示：
```go
// 發布消息
func PublishOrderCreated(order *Order) error {
    // 發送到消息隊列
}

// 訂閱消息
func SubscribeOrderEvents() {
    // 監聽訂單事件
}
```

---

下一章：[Chapter 53: gRPC 與 Protocol Buffers](53-grpc.md)
