# Chapter 59: 生產環境最佳實踐

## 概述

將 Go 應用部署到生產環境需要考慮日誌、監控、錯誤處理、優雅關閉等多個方面。本章將介紹 Go 生產環境的最佳實踐，並與 Node.js 的生產環境實踐進行對比。

## Node.js vs Go 生產環境對比

### Node.js 生產環境工具
```
- PM2（進程管理）
- Winston/Bunyan（日誌）
- Sentry（錯誤追蹤）
- New Relic/DataDog（監控）
- dotenv（環境變量）
```

### Go 生產環境工具
```
- Systemd（進程管理）
- Zap/Logrus（日誌）
- Sentry（錯誤追蹤）
- Prometheus（監控）
- Viper（配置管理）
```

## 結構化日誌

### 1. 使用 Zap（高性能日誌庫）

#### 安裝
```bash
go get -u go.uber.org/zap
```

#### 基本使用

```go
package main

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func main() {
    // 開發環境配置
    logger, _ := zap.NewDevelopment()
    defer logger.Sync()

    logger.Info("Failed to fetch URL",
        zap.String("url", "http://example.com"),
        zap.Int("attempt", 3),
        zap.Duration("backoff", time.Second),
    )
}
```

#### 生產環境配置

```go
package logger

import (
    "os"

    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func NewProductionLogger() (*zap.Logger, error) {
    config := zap.NewProductionEncoderConfig()
    config.TimeKey = "timestamp"
    config.EncodeTime = zapcore.ISO8601TimeEncoder

    fileEncoder := zapcore.NewJSONEncoder(config)
    consoleEncoder := zapcore.NewConsoleEncoder(config)

    // 寫入文件
    logFile, _ := os.OpenFile("app.log", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)

    // 多輸出
    core := zapcore.NewTee(
        zapcore.NewCore(fileEncoder, zapcore.AddSync(logFile), zapcore.InfoLevel),
        zapcore.NewCore(consoleEncoder, zapcore.AddSync(os.Stdout), zapcore.DebugLevel),
    )

    logger := zap.New(core, zap.AddCaller(), zap.AddStacktrace(zapcore.ErrorLevel))
    return logger, nil
}
```

#### 在應用中使用

```go
package main

import (
    "myapp/pkg/logger"
    "go.uber.org/zap"
)

var log *zap.Logger

func init() {
    var err error
    log, err = logger.NewProductionLogger()
    if err != nil {
        panic(err)
    }
}

func main() {
    defer log.Sync()

    log.Info("Application started",
        zap.String("version", "1.0.0"),
        zap.Int("port", 8080),
    )

    // 業務邏輯
    if err := doSomething(); err != nil {
        log.Error("Failed to do something",
            zap.Error(err),
            zap.String("context", "main"),
        )
    }
}
```

### 2. 使用 Logrus

```go
package main

import (
    "os"

    "github.com/sirupsen/logrus"
)

func main() {
    log := logrus.New()

    // 設置輸出格式
    log.SetFormatter(&logrus.JSONFormatter{})

    // 設置輸出目標
    file, err := os.OpenFile("app.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
    if err == nil {
        log.SetOutput(file)
    } else {
        log.Info("Failed to log to file, using default stderr")
    }

    // 設置日誌級別
    log.SetLevel(logrus.InfoLevel)

    // 使用
    log.WithFields(logrus.Fields{
        "event": "user_login",
        "user_id": 123,
    }).Info("User logged in")
}
```

### 3. HTTP 請求日誌中間件

```go
package middleware

import (
    "net/http"
    "time"

    "go.uber.org/zap"
)

type responseWriter struct {
    http.ResponseWriter
    status int
    size   int
}

func (rw *responseWriter) WriteHeader(status int) {
    rw.status = status
    rw.ResponseWriter.WriteHeader(status)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    size, err := rw.ResponseWriter.Write(b)
    rw.size += size
    return size, err
}

func RequestLogger(logger *zap.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()

            rw := &responseWriter{
                ResponseWriter: w,
                status:         200,
            }

            next.ServeHTTP(rw, r)

            logger.Info("HTTP request",
                zap.String("method", r.Method),
                zap.String("path", r.URL.Path),
                zap.String("remote_addr", r.RemoteAddr),
                zap.Int("status", rw.status),
                zap.Int("size", rw.size),
                zap.Duration("duration", time.Since(start)),
                zap.String("user_agent", r.UserAgent()),
            )
        })
    }
}
```

## 優雅關閉（Graceful Shutdown）

### 基本實現

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    // 創建服務器
    server := &http.Server{
        Addr:    ":8080",
        Handler: http.HandlerFunc(handler),
    }

    // 啟動服務器
    go func() {
        log.Println("Server starting on :8080")
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("Server failed: %v", err)
        }
    }()

    // 等待中斷信號
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Server shutting down...")

    // 優雅關閉（最多等待 30 秒）
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatalf("Server forced to shutdown: %v", err)
    }

    log.Println("Server exited")
}

func handler(w http.ResponseWriter, r *http.Request) {
    // 模擬長時間處理
    time.Sleep(2 * time.Second)
    w.Write([]byte("OK"))
}
```

### 完整的優雅關閉

```go
package main

import (
    "context"
    "database/sql"
    "log"
    "net/http"
    "os"
    "os/signal"
    "sync"
    "syscall"
    "time"
)

type App struct {
    server *http.Server
    db     *sql.DB
    // 其他資源
}

func (a *App) Shutdown(ctx context.Context) error {
    var wg sync.WaitGroup
    errChan := make(chan error, 2)

    // 關閉 HTTP 服務器
    wg.Add(1)
    go func() {
        defer wg.Done()
        log.Println("Shutting down HTTP server...")
        if err := a.server.Shutdown(ctx); err != nil {
            errChan <- err
        }
    }()

    // 關閉數據庫連接
    wg.Add(1)
    go func() {
        defer wg.Done()
        log.Println("Closing database connections...")
        if err := a.db.Close(); err != nil {
            errChan <- err
        }
    }()

    // 等待所有資源關閉
    done := make(chan struct{})
    go func() {
        wg.Wait()
        close(done)
    }()

    select {
    case <-done:
        log.Println("All resources closed gracefully")
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func main() {
    app := &App{
        server: &http.Server{Addr: ":8080"},
        // 初始化其他資源
    }

    // 啟動服務器
    go func() {
        if err := app.server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // 等待信號
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    // 優雅關閉
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := app.Shutdown(ctx); err != nil {
        log.Fatalf("Shutdown failed: %v", err)
    }

    log.Println("Application stopped")
}
```

## 錯誤處理和恢復

### 1. Panic 恢復中間件

```go
package middleware

import (
    "net/http"
    "runtime/debug"

    "go.uber.org/zap"
)

func Recovery(logger *zap.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            defer func() {
                if err := recover(); err != nil {
                    logger.Error("Panic recovered",
                        zap.Any("error", err),
                        zap.String("stack", string(debug.Stack())),
                        zap.String("path", r.URL.Path),
                    )

                    http.Error(w, "Internal Server Error", http.StatusInternalServerError)
                }
            }()

            next.ServeHTTP(w, r)
        })
    }
}
```

### 2. 錯誤追蹤（Sentry）

```go
package main

import (
    "net/http"
    "time"

    "github.com/getsentry/sentry-go"
    sentryhttp "github.com/getsentry/sentry-go/http"
)

func main() {
    // 初始化 Sentry
    err := sentry.Init(sentry.ClientOptions{
        Dsn:              "your-dsn-here",
        Environment:      "production",
        Release:          "myapp@1.0.0",
        TracesSampleRate: 1.0,
    })
    if err != nil {
        log.Fatal(err)
    }
    defer sentry.Flush(2 * time.Second)

    // 使用 Sentry 中間件
    sentryHandler := sentryhttp.New(sentryhttp.Options{})
    http.Handle("/", sentryHandler.Handle(http.HandlerFunc(handler)))

    http.ListenAndServe(":8080", nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
    if hub := sentry.GetHubFromContext(r.Context()); hub != nil {
        hub.WithScope(func(scope *sentry.Scope) {
            scope.SetTag("handler", "main")
            scope.SetUser(sentry.User{
                ID:       "123",
                Username: "alice",
            })

            // 捕獲錯誤
            if err := doSomething(); err != nil {
                hub.CaptureException(err)
            }
        })
    }
}
```

## 配置管理

### 使用 Viper

```go
package config

import (
    "github.com/spf13/viper"
)

type Config struct {
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
    Redis    RedisConfig    `mapstructure:"redis"`
    Log      LogConfig      `mapstructure:"log"`
}

type ServerConfig struct {
    Port int    `mapstructure:"port"`
    Host string `mapstructure:"host"`
}

type DatabaseConfig struct {
    Host     string `mapstructure:"host"`
    Port     int    `mapstructure:"port"`
    User     string `mapstructure:"user"`
    Password string `mapstructure:"password"`
    DBName   string `mapstructure:"dbname"`
}

type RedisConfig struct {
    Host string `mapstructure:"host"`
    Port int    `mapstructure:"port"`
}

type LogConfig struct {
    Level  string `mapstructure:"level"`
    Format string `mapstructure:"format"`
}

func Load() (*Config, error) {
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath(".")
    viper.AddConfigPath("./config")

    // 環境變量
    viper.AutomaticEnv()
    viper.SetEnvPrefix("APP")

    // 默認值
    viper.SetDefault("server.port", 8080)
    viper.SetDefault("log.level", "info")

    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }

    var config Config
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }

    return &config, nil
}
```

### config.yaml

```yaml
server:
  port: 8080
  host: 0.0.0.0

database:
  host: localhost
  port: 5432
  user: postgres
  password: ${DB_PASSWORD}
  dbname: myapp

redis:
  host: localhost
  port: 6379

log:
  level: info
  format: json
```

## 健康檢查

### 基本健康檢查

```go
package handlers

import (
    "database/sql"
    "encoding/json"
    "net/http"
)

type HealthHandler struct {
    db *sql.DB
}

type HealthResponse struct {
    Status   string            `json:"status"`
    Checks   map[string]string `json:"checks"`
    Version  string            `json:"version"`
    Uptime   string            `json:"uptime"`
}

func (h *HealthHandler) Health(w http.ResponseWriter, r *http.Request) {
    checks := make(map[string]string)

    // 檢查數據庫
    if err := h.db.Ping(); err != nil {
        checks["database"] = "unhealthy"
        w.WriteHeader(http.StatusServiceUnavailable)
    } else {
        checks["database"] = "healthy"
    }

    // 檢查其他依賴...

    status := "healthy"
    for _, check := range checks {
        if check != "healthy" {
            status = "unhealthy"
            break
        }
    }

    response := HealthResponse{
        Status:  status,
        Checks:  checks,
        Version: "1.0.0",
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

### 深度健康檢查

```go
package healthcheck

import (
    "context"
    "database/sql"
    "time"
)

type Checker interface {
    Check(ctx context.Context) error
}

type DatabaseChecker struct {
    db *sql.DB
}

func (dc *DatabaseChecker) Check(ctx context.Context) error {
    ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()

    return dc.db.PingContext(ctx)
}

type RedisChecker struct {
    // Redis client
}

func (rc *RedisChecker) Check(ctx context.Context) error {
    // 檢查 Redis 連接
    return nil
}

type HealthChecker struct {
    checkers map[string]Checker
}

func NewHealthChecker() *HealthChecker {
    return &HealthChecker{
        checkers: make(map[string]Checker),
    }
}

func (hc *HealthChecker) AddChecker(name string, checker Checker) {
    hc.checkers[name] = checker
}

func (hc *HealthChecker) CheckAll(ctx context.Context) map[string]error {
    results := make(map[string]error)

    for name, checker := range hc.checkers {
        results[name] = checker.Check(ctx)
    }

    return results
}
```

## 監控和指標

### Prometheus 集成

```go
package main

import (
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "endpoint"},
    )

    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
)

func PrometheusMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        activeConnections.Inc()
        defer activeConnections.Dec()

        start := time.Now()

        rw := &responseWriter{ResponseWriter: w, status: 200}
        next.ServeHTTP(rw, r)

        duration := time.Since(start).Seconds()

        httpRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            http.StatusText(rw.status),
        ).Inc()

        httpRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.Handle("/", PrometheusMiddleware(http.HandlerFunc(handler)))
    http.ListenAndServe(":8080", nil)
}
```

## 速率限制

### 基於 Token Bucket

```go
package middleware

import (
    "net/http"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

type RateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, b int) *RateLimiter {
    return &RateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    b,
    }
}

func (rl *RateLimiter) getLimiter(key string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    limiter, exists := rl.limiters[key]
    if !exists {
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.limiters[key] = limiter
    }

    return limiter
}

func (rl *RateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 使用 IP 作為限流鍵
        key := r.RemoteAddr

        limiter := rl.getLimiter(key)

        if !limiter.Allow() {
            http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

## Node.js 生產環境對比

### PM2 配置

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: './app.js',
    instances: 'max',  // 使用所有 CPU 核心
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    error_file: './logs/err.log',
    out_file: './logs/out.log',
    log_date_format: 'YYYY-MM-DD HH:mm:ss Z',
    max_memory_restart: '1G'
  }]
};
```

### 優雅關閉（Node.js）

```javascript
const express = require('express');
const app = express();

const server = app.listen(3000);

process.on('SIGTERM', () => {
    console.log('SIGTERM signal received: closing HTTP server');
    server.close(() => {
        console.log('HTTP server closed');
        // 關閉其他資源
        process.exit(0);
    });
});
```

## 重點總結

### Go 生產環境優勢

1. **單一二進制**
   - 部署簡單
   - 無運行時依賴

2. **性能優異**
   - 低內存占用
   - 高並發處理

3. **內置工具**
   - pprof 性能分析
   - race 檢測器

### 最佳實踐清單

- [ ] 使用結構化日誌（Zap/Logrus）
- [ ] 實現優雅關閉
- [ ] 添加健康檢查端點
- [ ] 集成監控（Prometheus）
- [ ] 錯誤追蹤（Sentry）
- [ ] 配置管理（Viper）
- [ ] 速率限制
- [ ] Panic 恢復
- [ ] 請求日誌
- [ ] 版本信息

## 練習題

### 練習 1: 完善日誌系統

實現一個完整的日誌系統，包含：
- 不同級別的日誌
- 日誌輪轉
- 結構化輸出

### 練習 2: 實現健康檢查

創建健康檢查端點，檢查：
- 數據庫連接
- Redis 連接
- 磁盤空間
- 內存使用

### 練習 3: 添加監控

使用 Prometheus 監控：
- 請求數量和延遲
- 錯誤率
- Goroutine 數量
- 內存使用

### 練習 4: 優雅關閉

實現完整的優雅關閉，包括：
- HTTP 服務器
- 數據庫連接
- 後台任務
- 清理臨時資源

### 練習 5: 配置管理

使用 Viper 實現：
- 多環境配置
- 環境變量覆蓋
- 熱重載配置

---

下一章：[Chapter 60: 從 Node.js 遷移到 Go](60-migration-strategy.md)
