# Chapter 43: PostgreSQL 集成

## 概述

PostgreSQL 是功能最強大的開源關聯式數據庫之一，Go 通過 `lib/pq` 和 `pgx` 驅動提供了完整的 PostgreSQL 支持。本章將深入探討 PostgreSQL 特有功能，如 JSON、數組、全文搜索等。

## Node.js vs Go 對比

### Node.js (node-postgres/pg)
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'postgres',
  password: 'password',
  max: 20,
  idleTimeoutMillis: 30000
});

// 查詢
const result = await pool.query('SELECT * FROM users WHERE id = $1', [1]);
console.log(result.rows[0]);

// JSON 查詢
const jsonResult = await pool.query(
  "SELECT * FROM products WHERE data->>'category' = $1",
  ['electronics']
);

// 事務
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('INSERT INTO users(name) VALUES($1)', ['Alice']);
  await client.query('COMMIT');
} catch (e) {
  await client.query('ROLLBACK');
  throw e;
} finally {
  client.release();
}
```

### Go (lib/pq)
```go
import (
    "database/sql"
    _ "github.com/lib/pq"
)

// 連接
connStr := "host=localhost port=5432 user=postgres password=password dbname=mydb sslmode=disable"
db, err := sql.Open("postgres", connStr)

// 查詢
var user User
err = db.QueryRow("SELECT * FROM users WHERE id = $1", 1).Scan(&user.ID, &user.Name)

// JSON 查詢
rows, err := db.Query("SELECT * FROM products WHERE data->>'category' = $1", "electronics")

// 事務
tx, err := db.Begin()
defer func() {
    if err != nil {
        tx.Rollback()
    } else {
        tx.Commit()
    }
}()
_, err = tx.Exec("INSERT INTO users(name) VALUES($1)", "Alice")
```

## 安裝驅動

```bash
# lib/pq - 純 Go 實現
go get github.com/lib/pq

# pgx - 高性能驅動
go get github.com/jackc/pgx/v5
go get github.com/jackc/pgx/v5/stdlib
```

## 1. 基礎連接與配置

### 使用 lib/pq

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "time"

    _ "github.com/lib/pq"
)

func main() {
    // 方式 1: 連接字符串
    connStr := "postgres://postgres:password@localhost:5432/mydb?sslmode=disable"
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // 方式 2: 參數格式
    connStr2 := "host=localhost port=5432 user=postgres password=password dbname=mydb sslmode=disable"
    db2, err := sql.Open("postgres", connStr2)
    if err != nil {
        log.Fatal(err)
    }
    defer db2.Close()

    // 配置連接池
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)

    // 測試連接
    if err := db.Ping(); err != nil {
        log.Fatal("無法連接數據庫:", err)
    }

    fmt.Println("成功連接到 PostgreSQL!")
}
```

### 使用 pgx（更高性能）

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/jackc/pgx/v5/pgxpool"
)

func main() {
    // 創建連接池配置
    config, err := pgxpool.ParseConfig(
        "postgres://postgres:password@localhost:5432/mydb")
    if err != nil {
        log.Fatal(err)
    }

    // 配置連接池
    config.MaxConns = 25
    config.MinConns = 5
    config.MaxConnLifetime = 5 * time.Minute

    // 創建連接池
    pool, err := pgxpool.NewWithConfig(context.Background(), config)
    if err != nil {
        log.Fatal(err)
    }
    defer pool.Close()

    // 測試連接
    if err := pool.Ping(context.Background()); err != nil {
        log.Fatal("無法連接數據庫:", err)
    }

    fmt.Println("成功連接到 PostgreSQL!")
}
```

**Node.js 對比：**
```javascript
const { Pool } = require('pg');

// 使用連接字符串
const pool1 = new Pool({
  connectionString: 'postgres://postgres:password@localhost:5432/mydb'
});

// 使用配置對象
const pool2 = new Pool({
  host: 'localhost',
  port: 5432,
  database: 'mydb',
  user: 'postgres',
  password: 'password',
  max: 25,
  min: 5,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

// 測試連接
try {
  const client = await pool1.connect();
  console.log('成功連接到 PostgreSQL!');
  client.release();
} catch (err) {
  console.error('無法連接數據庫:', err);
}
```

## 2. PostgreSQL 特有功能

### ARRAY 類型

```go
package main

import (
    "database/sql"
    "fmt"

    "github.com/lib/pq"
)

type Product struct {
    ID     int
    Name   string
    Tags   pq.StringArray  // PostgreSQL 數組類型
    Prices pq.Float64Array // 數字數組
}

func arrayExample(db *sql.DB) {
    // 創建表
    _, err := db.Exec(`
        CREATE TABLE IF NOT EXISTS products (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100),
            tags TEXT[],
            prices NUMERIC[]
        )
    `)

    // 插入帶數組的數據
    tags := pq.StringArray{"electronics", "gadget", "phone"}
    prices := pq.Float64Array{999.99, 899.99, 799.99}

    _, err = db.Exec(
        "INSERT INTO products(name, tags, prices) VALUES($1, $2, $3)",
        "iPhone", tags, prices,
    )

    // 查詢數組
    var product Product
    err = db.QueryRow("SELECT id, name, tags, prices FROM products WHERE id = $1", 1).
        Scan(&product.ID, &product.Name, &product.Tags, &product.Prices)

    fmt.Printf("Product: %+v\n", product)
    fmt.Printf("Tags: %v\n", []string(product.Tags))
    fmt.Printf("Prices: %v\n", []float64(product.Prices))

    // 數組查詢條件
    // 包含特定標籤
    rows, err := db.Query("SELECT * FROM products WHERE 'phone' = ANY(tags)")

    // 數組重疊
    searchTags := pq.StringArray{"phone", "laptop"}
    rows, err = db.Query("SELECT * FROM products WHERE tags && $1", searchTags)

    // 數組包含
    rows, err = db.Query("SELECT * FROM products WHERE tags @> $1",
        pq.StringArray{"phone"})
}
```

**Node.js 對比：**
```javascript
// Node.js - PostgreSQL 數組
// 插入
await pool.query(
  'INSERT INTO products(name, tags, prices) VALUES($1, $2, $3)',
  ['iPhone', ['electronics', 'gadget', 'phone'], [999.99, 899.99, 799.99]]
);

// 查詢
const result = await pool.query(
  'SELECT * FROM products WHERE id = $1',
  [1]
);
const product = result.rows[0];
console.log('Tags:', product.tags);  // JavaScript 數組

// 數組查詢
await pool.query("SELECT * FROM products WHERE 'phone' = ANY(tags)");
await pool.query("SELECT * FROM products WHERE tags && $1",
  [['phone', 'laptop']]);
```

### JSON/JSONB 類型

```go
package main

import (
    "database/sql"
    "encoding/json"
    "fmt"
)

type User struct {
    ID       int
    Name     string
    Metadata json.RawMessage  // JSON 數據
}

type UserMetadata struct {
    Age       int               `json:"age"`
    City      string            `json:"city"`
    Interests []string          `json:"interests"`
    Settings  map[string]string `json:"settings"`
}

func jsonExample(db *sql.DB) {
    // 創建表
    _, err := db.Exec(`
        CREATE TABLE IF NOT EXISTS users (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100),
            metadata JSONB
        )
    `)

    // 準備 JSON 數據
    metadata := UserMetadata{
        Age:  25,
        City: "Taipei",
        Interests: []string{"programming", "reading"},
        Settings: map[string]string{
            "theme": "dark",
            "language": "zh-TW",
        },
    }

    jsonData, _ := json.Marshal(metadata)

    // 插入 JSON 數據
    _, err = db.Exec(
        "INSERT INTO users(name, metadata) VALUES($1, $2)",
        "Alice", jsonData,
    )

    // 查詢 JSON 數據
    var user User
    err = db.QueryRow("SELECT id, name, metadata FROM users WHERE id = $1", 1).
        Scan(&user.ID, &user.Name, &user.Metadata)

    // 解析 JSON
    var meta UserMetadata
    json.Unmarshal(user.Metadata, &meta)
    fmt.Printf("User: %s, Age: %d, City: %s\n", user.Name, meta.Age, meta.City)

    // JSON 字段查詢
    // -> 返回 JSON 對象，->> 返回文本
    rows, err := db.Query("SELECT * FROM users WHERE metadata->>'city' = $1", "Taipei")

    // JSONB 包含
    rows, err = db.Query("SELECT * FROM users WHERE metadata @> $1",
        `{"city": "Taipei"}`)

    // JSON 數組查詢
    rows, err = db.Query(
        "SELECT * FROM users WHERE metadata->'interests' ? $1",
        "programming",
    )

    // 更新 JSON 字段
    _, err = db.Exec(
        "UPDATE users SET metadata = jsonb_set(metadata, '{age}', $1) WHERE id = $2",
        "26", 1,
    )

    // JSON 聚合
    type CityCount struct {
        City  string
        Count int
    }
    var results []CityCount
    rows, err = db.Query(`
        SELECT metadata->>'city' as city, COUNT(*) as count
        FROM users
        GROUP BY metadata->>'city'
    `)
    defer rows.Close()

    for rows.Next() {
        var r CityCount
        rows.Scan(&r.City, &r.Count)
        results = append(results, r)
    }
}
```

**Node.js 對比：**
```javascript
// Node.js - PostgreSQL JSON
// 插入
const metadata = {
  age: 25,
  city: 'Taipei',
  interests: ['programming', 'reading'],
  settings: { theme: 'dark', language: 'zh-TW' }
};

await pool.query(
  'INSERT INTO users(name, metadata) VALUES($1, $2)',
  ['Alice', JSON.stringify(metadata)]
);

// 查詢
const result = await pool.query(
  'SELECT * FROM users WHERE id = $1',
  [1]
);
const user = result.rows[0];
const meta = JSON.parse(user.metadata);  // 或直接是對象（取決於驅動）

// JSON 查詢
await pool.query("SELECT * FROM users WHERE metadata->>'city' = $1", ['Taipei']);
await pool.query("SELECT * FROM users WHERE metadata @> $1", ['{"city": "Taipei"}']);

// 更新 JSON
await pool.query(
  "UPDATE users SET metadata = jsonb_set(metadata, '{age}', $1) WHERE id = $2",
  ['"26"', 1]
);
```

### ENUM 類型

```go
package main

import (
    "database/sql"
)

type OrderStatus string

const (
    OrderStatusPending   OrderStatus = "pending"
    OrderStatusProcessing OrderStatus = "processing"
    OrderStatusShipped   OrderStatus = "shipped"
    OrderStatusDelivered OrderStatus = "delivered"
    OrderStatusCancelled OrderStatus = "cancelled"
)

func enumExample(db *sql.DB) {
    // 創建 ENUM 類型
    _, err := db.Exec(`
        DO $$ BEGIN
            CREATE TYPE order_status AS ENUM (
                'pending', 'processing', 'shipped', 'delivered', 'cancelled'
            );
        EXCEPTION
            WHEN duplicate_object THEN null;
        END $$;
    `)

    // 創建使用 ENUM 的表
    _, err = db.Exec(`
        CREATE TABLE IF NOT EXISTS orders (
            id SERIAL PRIMARY KEY,
            order_number VARCHAR(50),
            status order_status DEFAULT 'pending'
        )
    `)

    // 插入數據
    _, err = db.Exec(
        "INSERT INTO orders(order_number, status) VALUES($1, $2)",
        "ORD-001", OrderStatusPending,
    )

    // 查詢
    var orderNumber string
    var status OrderStatus
    err = db.QueryRow("SELECT order_number, status FROM orders WHERE id = $1", 1).
        Scan(&orderNumber, &status)

    fmt.Printf("Order: %s, Status: %s\n", orderNumber, status)

    // 更新狀態
    _, err = db.Exec(
        "UPDATE orders SET status = $1 WHERE order_number = $2",
        OrderStatusShipped, "ORD-001",
    )
}
```

**Node.js 對比：**
```javascript
// Node.js - PostgreSQL ENUM
// 創建 ENUM
await pool.query(`
  CREATE TYPE order_status AS ENUM (
    'pending', 'processing', 'shipped', 'delivered', 'cancelled'
  )
`);

// 插入
await pool.query(
  'INSERT INTO orders(order_number, status) VALUES($1, $2)',
  ['ORD-001', 'pending']
);

// 查詢
const result = await pool.query(
  'SELECT order_number, status FROM orders WHERE id = $1',
  [1]
);
console.log('Status:', result.rows[0].status);
```

## 3. 全文搜索

```go
package main

import (
    "database/sql"
    "strings"
)

type Article struct {
    ID      int
    Title   string
    Content string
    Rank    float64
}

func fullTextSearchExample(db *sql.DB) {
    // 創建表
    _, err := db.Exec(`
        CREATE TABLE IF NOT EXISTS articles (
            id SERIAL PRIMARY KEY,
            title VARCHAR(200),
            content TEXT,
            search_vector tsvector
        )
    `)

    // 創建全文搜索索引
    _, err = db.Exec(`
        CREATE INDEX IF NOT EXISTS idx_articles_search
        ON articles USING GIN(search_vector)
    `)

    // 創建觸發器自動更新搜索向量
    _, err = db.Exec(`
        CREATE OR REPLACE FUNCTION articles_search_trigger() RETURNS trigger AS $$
        BEGIN
            NEW.search_vector :=
                setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
                setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'B');
            RETURN NEW;
        END
        $$ LANGUAGE plpgsql;

        DROP TRIGGER IF EXISTS tsvector_update ON articles;
        CREATE TRIGGER tsvector_update
        BEFORE INSERT OR UPDATE ON articles
        FOR EACH ROW EXECUTE FUNCTION articles_search_trigger();
    `)

    // 插入文章
    articles := []struct {
        title   string
        content string
    }{
        {"Introduction to Go", "Go is a programming language designed by Google..."},
        {"PostgreSQL Guide", "PostgreSQL is a powerful open source database..."},
        {"Go and PostgreSQL", "Learn how to use Go with PostgreSQL database..."},
    }

    for _, article := range articles {
        _, err = db.Exec(
            "INSERT INTO articles(title, content) VALUES($1, $2)",
            article.title, article.content,
        )
    }

    // 全文搜索
    keyword := "Go programming"
    query := strings.Join(strings.Fields(keyword), " & ")

    rows, err := db.Query(`
        SELECT id, title, content,
               ts_rank(search_vector, to_tsquery('english', $1)) as rank
        FROM articles
        WHERE search_vector @@ to_tsquery('english', $1)
        ORDER BY rank DESC
    `, query)
    defer rows.Close()

    var results []Article
    for rows.Next() {
        var article Article
        rows.Scan(&article.ID, &article.Title, &article.Content, &article.Rank)
        results = append(results, article)
    }

    fmt.Printf("找到 %d 篇文章\n", len(results))
    for _, article := range results {
        fmt.Printf("- %s (相關度: %.4f)\n", article.Title, article.Rank)
    }

    // 高亮搜索結果
    rows, err = db.Query(`
        SELECT id, title,
               ts_headline('english', content, to_tsquery('english', $1)) as snippet
        FROM articles
        WHERE search_vector @@ to_tsquery('english', $1)
    `, query)
}
```

**Node.js 對比：**
```javascript
// Node.js - PostgreSQL 全文搜索
// 搜索
const keyword = 'Go programming';
const query = keyword.split(/\s+/).join(' & ');

const result = await pool.query(`
  SELECT id, title, content,
         ts_rank(search_vector, to_tsquery('english', $1)) as rank
  FROM articles
  WHERE search_vector @@ to_tsquery('english', $1)
  ORDER BY rank DESC
`, [query]);

console.log(`找到 ${result.rows.length} 篇文章`);
result.rows.forEach(article => {
  console.log(`- ${article.title} (相關度: ${article.rank})`);
});
```

## 4. 高級查詢技巧

### CTE (Common Table Expressions)

```go
package main

import (
    "database/sql"
)

type Employee struct {
    ID         int
    Name       string
    ManagerID  sql.NullInt64
    Department string
    Level      int
}

func cteExample(db *sql.DB) {
    // 遞歸 CTE - 查詢組織層級
    rows, err := db.Query(`
        WITH RECURSIVE org_tree AS (
            -- 基礎查詢：頂層員工
            SELECT id, name, manager_id, department, 1 as level
            FROM employees
            WHERE manager_id IS NULL

            UNION ALL

            -- 遞歸部分：下屬員工
            SELECT e.id, e.name, e.manager_id, e.department, ot.level + 1
            FROM employees e
            INNER JOIN org_tree ot ON e.manager_id = ot.id
        )
        SELECT * FROM org_tree
        ORDER BY level, name
    `)
    defer rows.Close()

    var employees []Employee
    for rows.Next() {
        var emp Employee
        rows.Scan(&emp.ID, &emp.Name, &emp.ManagerID, &emp.Department, &emp.Level)
        employees = append(employees, emp)
    }

    // 非遞歸 CTE - 多個臨時結果集
    rows, err = db.Query(`
        WITH
            sales_summary AS (
                SELECT product_id, SUM(amount) as total_sales
                FROM orders
                GROUP BY product_id
            ),
            product_info AS (
                SELECT id, name, category
                FROM products
                WHERE active = true
            )
        SELECT p.name, p.category, s.total_sales
        FROM product_info p
        LEFT JOIN sales_summary s ON p.id = s.product_id
        ORDER BY s.total_sales DESC
    `)
}
```

### Window Functions

```go
package main

import (
    "database/sql"
)

type SalesData struct {
    Date       string
    Amount     float64
    RunningSum float64
    Rank       int
    RowNumber  int
}

func windowFunctionExample(db *sql.DB) {
    // 運行總和、排名、行號
    rows, err := db.Query(`
        SELECT
            date,
            amount,
            SUM(amount) OVER (ORDER BY date) as running_sum,
            RANK() OVER (ORDER BY amount DESC) as rank,
            ROW_NUMBER() OVER (ORDER BY date) as row_number
        FROM sales
        ORDER BY date
    `)
    defer rows.Close()

    var results []SalesData
    for rows.Next() {
        var data SalesData
        rows.Scan(&data.Date, &data.Amount, &data.RunningSum, &data.Rank, &data.RowNumber)
        results = append(results, data)
    }

    // 按分區計算
    rows, err = db.Query(`
        SELECT
            product_id,
            date,
            amount,
            AVG(amount) OVER (PARTITION BY product_id) as avg_by_product,
            MAX(amount) OVER (PARTITION BY product_id) as max_by_product
        FROM sales
        ORDER BY product_id, date
    `)

    // LAG 和 LEAD - 訪問前後行
    rows, err = db.Query(`
        SELECT
            date,
            amount,
            LAG(amount, 1) OVER (ORDER BY date) as prev_amount,
            LEAD(amount, 1) OVER (ORDER BY date) as next_amount,
            amount - LAG(amount, 1) OVER (ORDER BY date) as change
        FROM sales
        ORDER BY date
    `)
}
```

**Node.js 對比：**
```javascript
// Node.js - Window Functions
const result = await pool.query(`
  SELECT
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) as running_sum,
    RANK() OVER (ORDER BY amount DESC) as rank,
    ROW_NUMBER() OVER (ORDER BY date) as row_number
  FROM sales
  ORDER BY date
`);

result.rows.forEach(row => {
  console.log(`${row.date}: ${row.amount} (累計: ${row.running_sum})`);
});
```

## 5. 事務與並發控制

### 隔離級別

```go
package main

import (
    "context"
    "database/sql"
)

func transactionIsolationExample(db *sql.DB) {
    ctx := context.Background()

    // Read Committed（默認）
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelReadCommitted,
    })

    // Repeatable Read
    tx, err = db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelRepeatableRead,
    })

    // Serializable
    tx, err = db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })

    defer func() {
        if err != nil {
            tx.Rollback()
        } else {
            tx.Commit()
        }
    }()

    // 執行事務操作
    _, err = tx.Exec("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
}
```

### 行鎖

```go
package main

import (
    "database/sql"
)

func rowLockingExample(db *sql.DB) {
    tx, err := db.Begin()
    if err != nil {
        return
    }
    defer tx.Rollback()

    // SELECT FOR UPDATE - 獨占鎖
    var balance float64
    err = tx.QueryRow(
        "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE",
        1,
    ).Scan(&balance)

    if balance >= 100 {
        // 執行扣款
        _, err = tx.Exec(
            "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
            100, 1,
        )
    }

    tx.Commit()

    // SELECT FOR SHARE - 共享鎖
    err = tx.QueryRow(
        "SELECT balance FROM accounts WHERE id = $1 FOR SHARE",
        1,
    ).Scan(&balance)

    // NOWAIT - 不等待鎖
    err = tx.QueryRow(
        "SELECT balance FROM accounts WHERE id = $1 FOR UPDATE NOWAIT",
        1,
    ).Scan(&balance)
    if err != nil {
        fmt.Println("資源被鎖定")
    }
}
```

## 完整示例：電商系統

```go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "time"

    _ "github.com/lib/pq"
    "github.com/lib/pq"
)

// 模型定義
type Product struct {
    ID          int
    Name        string
    Description string
    Price       float64
    Stock       int
    Tags        pq.StringArray
    Attributes  json.RawMessage
    CreatedAt   time.Time
}

type Order struct {
    ID          int
    UserID      int
    TotalAmount float64
    Status      string
    Items       []OrderItem
    CreatedAt   time.Time
}

type OrderItem struct {
    ProductID int
    Quantity  int
    Price     float64
}

// Repository
type EcommerceRepository struct {
    db *sql.DB
}

func NewEcommerceRepository(db *sql.DB) *EcommerceRepository {
    return &EcommerceRepository{db: db}
}

// 搜索商品
func (r *EcommerceRepository) SearchProducts(keyword string, tags []string, minPrice, maxPrice float64) ([]Product, error) {
    query := `
        SELECT id, name, description, price, stock, tags, attributes
        FROM products
        WHERE
            ($1 = '' OR name ILIKE '%' || $1 || '%' OR description ILIKE '%' || $1 || '%')
            AND ($2::text[] IS NULL OR tags && $2)
            AND price BETWEEN $3 AND $4
            AND stock > 0
        ORDER BY
            CASE WHEN name ILIKE '%' || $1 || '%' THEN 1 ELSE 2 END,
            price
    `

    var tagArray pq.StringArray
    if len(tags) > 0 {
        tagArray = tags
    }

    rows, err := r.db.Query(query, keyword, tagArray, minPrice, maxPrice)
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var products []Product
    for rows.Next() {
        var p Product
        err := rows.Scan(&p.ID, &p.Name, &p.Description, &p.Price, &p.Stock, &p.Tags, &p.Attributes)
        if err != nil {
            return nil, err
        }
        products = append(products, p)
    }

    return products, nil
}

// 創建訂單（帶事務）
func (r *EcommerceRepository) CreateOrder(ctx context.Context, userID int, items []OrderItem) (*Order, error) {
    tx, err := r.db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    if err != nil {
        return nil, err
    }
    defer tx.Rollback()

    var totalAmount float64

    // 驗證庫存並計算總金額
    for _, item := range items {
        var price float64
        var stock int

        // 鎖定商品行
        err = tx.QueryRow(
            "SELECT price, stock FROM products WHERE id = $1 FOR UPDATE",
            item.ProductID,
        ).Scan(&price, &stock)

        if err != nil {
            return nil, fmt.Errorf("商品 %d 不存在", item.ProductID)
        }

        if stock < item.Quantity {
            return nil, fmt.Errorf("商品 %d 庫存不足", item.ProductID)
        }

        totalAmount += price * float64(item.Quantity)

        // 扣減庫存
        _, err = tx.Exec(
            "UPDATE products SET stock = stock - $1 WHERE id = $2",
            item.Quantity, item.ProductID,
        )
        if err != nil {
            return nil, err
        }
    }

    // 創建訂單
    var orderID int
    err = tx.QueryRow(
        "INSERT INTO orders(user_id, total_amount, status) VALUES($1, $2, $3) RETURNING id",
        userID, totalAmount, "pending",
    ).Scan(&orderID)

    if err != nil {
        return nil, err
    }

    // 創建訂單項
    for _, item := range items {
        var price float64
        tx.QueryRow("SELECT price FROM products WHERE id = $1", item.ProductID).Scan(&price)

        _, err = tx.Exec(
            "INSERT INTO order_items(order_id, product_id, quantity, price) VALUES($1, $2, $3, $4)",
            orderID, item.ProductID, item.Quantity, price,
        )
        if err != nil {
            return nil, err
        }
    }

    // 提交事務
    if err = tx.Commit(); err != nil {
        return nil, err
    }

    // 返回訂單信息
    order := &Order{
        ID:          orderID,
        UserID:      userID,
        TotalAmount: totalAmount,
        Status:      "pending",
        Items:       items,
        CreatedAt:   time.Now(),
    }

    return order, nil
}

// 獲取銷售統計（使用 Window Function）
func (r *EcommerceRepository) GetSalesStats(days int) (interface{}, error) {
    query := `
        WITH daily_sales AS (
            SELECT
                DATE(created_at) as date,
                SUM(total_amount) as amount
            FROM orders
            WHERE created_at >= NOW() - INTERVAL '%d days'
            GROUP BY DATE(created_at)
        )
        SELECT
            date,
            amount,
            SUM(amount) OVER (ORDER BY date) as cumulative,
            AVG(amount) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as moving_avg
        FROM daily_sales
        ORDER BY date
    `

    rows, err := r.db.Query(fmt.Sprintf(query, days))
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    type DailyStat struct {
        Date       string  `json:"date"`
        Amount     float64 `json:"amount"`
        Cumulative float64 `json:"cumulative"`
        MovingAvg  float64 `json:"moving_avg"`
    }

    var stats []DailyStat
    for rows.Next() {
        var stat DailyStat
        rows.Scan(&stat.Date, &stat.Amount, &stat.Cumulative, &stat.MovingAvg)
        stats = append(stats, stat)
    }

    return stats, nil
}

func main() {
    // 連接數據庫
    connStr := "postgres://postgres:password@localhost/ecommerce?sslmode=disable"
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        panic(err)
    }
    defer db.Close()

    repo := NewEcommerceRepository(db)

    // 搜索商品
    products, _ := repo.SearchProducts(
        "laptop",
        []string{"electronics"},
        500.0,
        2000.0,
    )
    fmt.Printf("找到 %d 個商品\n", len(products))

    // 創建訂單
    order, err := repo.CreateOrder(context.Background(), 1, []OrderItem{
        {ProductID: 1, Quantity: 2},
        {ProductID: 2, Quantity: 1},
    })
    if err != nil {
        fmt.Println("創建訂單失敗:", err)
    } else {
        fmt.Printf("訂單創建成功: #%d, 總金額: %.2f\n", order.ID, order.TotalAmount)
    }

    // 獲取銷售統計
    stats, _ := repo.GetSalesStats(30)
    fmt.Printf("銷售統計: %+v\n", stats)
}
```

## 重點總結

### PostgreSQL vs MySQL

| 特性 | PostgreSQL | MySQL |
|------|-----------|-------|
| **數組** | 原生支持 | 需要序列化 |
| **JSON** | JSONB（高效） | JSON（較慢） |
| **全文搜索** | 內置 | 基礎支持 |
| **Window Functions** | 完整支持 | 部分支持 |
| **CTE** | 支持遞歸 | 8.0+ 支持 |
| **並發控制** | MVCC | 鎖機制 |

### 最佳實踐

1. **使用 JSONB 而不是 JSON**（性能更好）
2. **為 JSONB 字段創建 GIN 索引**
3. **使用數組類型存儲列表數據**
4. **利用 CTE 簡化複雜查詢**
5. **合理使用 Window Functions 進行分析**
6. **事務中使用 FOR UPDATE 避免競態**

## 練習題

1. **基礎**：創建一個支持標籤搜索的文章系統（使用 ARRAY）
2. **中級**：實現一個用戶配置系統（使用 JSONB）
3. **高級**：實現組織架構樹查詢（使用遞歸 CTE）
4. **實戰**：構建一個完整的訂單系統，包含庫存控制、事務處理和銷售分析

## 下一步

下一章將學習 MongoDB 與 Go 的集成，探索 NoSQL 數據庫在 Go 中的使用。
