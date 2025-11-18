# Chapter 41: database/sql 標準接口

## 概述

`database/sql` 是 Go 標準庫提供的數據庫操作接口，它定義了一套統一的 API 來操作各種 SQL 數據庫。與 Node.js 中每個數據庫都有自己的驅動不同，Go 通過 `database/sql` 提供了一個標準化的抽象層。

## Node.js vs Go 對比

### Node.js 數據庫驅動
```javascript
// Node.js - 每個數據庫有獨立的 API
const mysql = require('mysql2/promise');
const { Pool } = require('pg');

// MySQL
const mysqlConnection = await mysql.createConnection({
  host: 'localhost',
  user: 'root',
  password: 'password',
  database: 'mydb'
});

// PostgreSQL - 完全不同的 API
const pgPool = new Pool({
  host: 'localhost',
  user: 'postgres',
  password: 'password',
  database: 'mydb'
});

// 查詢方式不同
const [mysqlRows] = await mysqlConnection.execute('SELECT * FROM users');
const pgResult = await pgPool.query('SELECT * FROM users');
```

### Go database/sql
```go
// Go - 統一的 database/sql 接口
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql"
    _ "github.com/lib/pq"
)

// MySQL
mysqlDB, err := sql.Open("mysql",
    "root:password@tcp(localhost:3306)/mydb")

// PostgreSQL - 相同的 API
pgDB, err := sql.Open("postgres",
    "host=localhost user=postgres password=password dbname=mydb")

// 查詢方式完全相同
mysqlRows, err := mysqlDB.Query("SELECT * FROM users")
pgRows, err := pgDB.Query("SELECT * FROM users")
```

## 核心概念

### 1. sql.DB - 連接池

`sql.DB` 不是單一連接，而是連接池管理器。

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "time"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    // 創建連接池
    db, err := sql.Open("mysql",
        "root:password@tcp(localhost:3306)/mydb")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // 配置連接池
    db.SetMaxOpenConns(25)                 // 最大打開連接數
    db.SetMaxIdleConns(5)                  // 最大空閒連接數
    db.SetConnMaxLifetime(5 * time.Minute) // 連接最大存活時間
    db.SetConnMaxIdleTime(10 * time.Minute)// 連接最大空閒時間

    // 測試連接
    if err := db.Ping(); err != nil {
        log.Fatal("無法連接數據庫:", err)
    }

    fmt.Println("數據庫連接成功!")
}
```

**Node.js 對比：**
```javascript
// Node.js - mysql2
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  password: 'password',
  database: 'mydb',
  waitForConnections: true,
  connectionLimit: 25,        // 類似 MaxOpenConns
  maxIdle: 5,                 // 類似 MaxIdleConns
  idleTimeout: 600000,        // 類似 ConnMaxIdleTime
  queueLimit: 0
});

// 測試連接
try {
  const connection = await pool.getConnection();
  console.log('數據庫連接成功!');
  connection.release();
} catch (error) {
  console.error('無法連接數據庫:', error);
}
```

### 2. 查詢操作

#### Query vs QueryRow vs Exec

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
)

type User struct {
    ID    int
    Name  string
    Email string
    Age   int
}

func main() {
    db, _ := sql.Open("mysql",
        "root:password@tcp(localhost:3306)/mydb")
    defer db.Close()

    // 1. Query - 返回多行
    rows, err := db.Query("SELECT id, name, email, age FROM users WHERE age > ?", 18)
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var u User
        err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.Age)
        if err != nil {
            log.Fatal(err)
        }
        users = append(users, u)
    }

    // 檢查迭代錯誤
    if err = rows.Err(); err != nil {
        log.Fatal(err)
    }

    fmt.Printf("找到 %d 個用戶\n", len(users))

    // 2. QueryRow - 返回單行
    var user User
    err = db.QueryRow("SELECT id, name, email, age FROM users WHERE id = ?", 1).
        Scan(&user.ID, &user.Name, &user.Email, &user.Age)

    if err == sql.ErrNoRows {
        fmt.Println("用戶不存在")
    } else if err != nil {
        log.Fatal(err)
    } else {
        fmt.Printf("用戶: %+v\n", user)
    }

    // 3. Exec - 執行不返回行的語句
    result, err := db.Exec("UPDATE users SET age = ? WHERE id = ?", 30, 1)
    if err != nil {
        log.Fatal(err)
    }

    rowsAffected, _ := result.RowsAffected()
    lastInsertId, _ := result.LastInsertId()

    fmt.Printf("影響行數: %d, 最後插入ID: %d\n", rowsAffected, lastInsertId)
}
```

**Node.js 對比：**
```javascript
// Node.js - mysql2
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: 'localhost',
  user: 'root',
  password: 'password',
  database: 'mydb'
});

async function databaseOperations() {
  // 1. 查詢多行
  const [rows] = await pool.execute(
    'SELECT id, name, email, age FROM users WHERE age > ?',
    [18]
  );
  console.log(`找到 ${rows.length} 個用戶`);

  // 2. 查詢單行
  const [users] = await pool.execute(
    'SELECT id, name, email, age FROM users WHERE id = ?',
    [1]
  );

  if (users.length === 0) {
    console.log('用戶不存在');
  } else {
    console.log('用戶:', users[0]);
  }

  // 3. 執行更新
  const [result] = await pool.execute(
    'UPDATE users SET age = ? WHERE id = ?',
    [30, 1]
  );

  console.log(`影響行數: ${result.affectedRows}, 最後插入ID: ${result.insertId}`);
}
```

### 3. 預處理語句 (Prepared Statements)

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
)

func main() {
    db, _ := sql.Open("mysql",
        "root:password@tcp(localhost:3306)/mydb")
    defer db.Close()

    // 創建預處理語句
    stmt, err := db.Prepare("INSERT INTO users(name, email, age) VALUES(?, ?, ?)")
    if err != nil {
        log.Fatal(err)
    }
    defer stmt.Close()

    // 多次執行同一語句
    users := []struct {
        name  string
        email string
        age   int
    }{
        {"Alice", "alice@example.com", 25},
        {"Bob", "bob@example.com", 30},
        {"Charlie", "charlie@example.com", 35},
    }

    for _, u := range users {
        result, err := stmt.Exec(u.name, u.email, u.age)
        if err != nil {
            log.Printf("插入失敗 %s: %v", u.name, err)
            continue
        }

        id, _ := result.LastInsertId()
        fmt.Printf("插入用戶 %s，ID: %d\n", u.name, id)
    }
}
```

**Node.js 對比：**
```javascript
// Node.js - mysql2
const pool = mysql.createPool({...});

async function insertUsers() {
  // 創建預處理語句
  const stmt = await pool.prepare(
    'INSERT INTO users(name, email, age) VALUES(?, ?, ?)'
  );

  const users = [
    {name: 'Alice', email: 'alice@example.com', age: 25},
    {name: 'Bob', email: 'bob@example.com', age: 30},
    {name: 'Charlie', email: 'charlie@example.com', age: 35}
  ];

  for (const u of users) {
    try {
      const [result] = await stmt.execute([u.name, u.email, u.age]);
      console.log(`插入用戶 ${u.name}，ID: ${result.insertId}`);
    } catch (error) {
      console.error(`插入失敗 ${u.name}:`, error);
    }
  }

  await stmt.close();
}
```

### 4. 事務處理

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
)

func transferMoney(db *sql.DB, fromUserID, toUserID int, amount float64) error {
    // 開始事務
    tx, err := db.Begin()
    if err != nil {
        return err
    }

    // defer 確保事務被提交或回滾
    defer func() {
        if err != nil {
            tx.Rollback()
            fmt.Println("事務回滾")
        }
    }()

    // 扣款
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance - ? WHERE user_id = ?",
        amount, fromUserID,
    )
    if err != nil {
        return fmt.Errorf("扣款失敗: %w", err)
    }

    // 入賬
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance + ? WHERE user_id = ?",
        amount, toUserID,
    )
    if err != nil {
        return fmt.Errorf("入賬失敗: %w", err)
    }

    // 記錄轉賬
    _, err = tx.Exec(
        "INSERT INTO transactions(from_user, to_user, amount) VALUES(?, ?, ?)",
        fromUserID, toUserID, amount,
    )
    if err != nil {
        return fmt.Errorf("記錄失敗: %w", err)
    }

    // 提交事務
    if err = tx.Commit(); err != nil {
        return fmt.Errorf("提交失敗: %w", err)
    }

    fmt.Println("事務提交成功")
    return nil
}

func main() {
    db, _ := sql.Open("mysql",
        "root:password@tcp(localhost:3306)/mydb")
    defer db.Close()

    err := transferMoney(db, 1, 2, 100.00)
    if err != nil {
        log.Printf("轉賬失敗: %v", err)
    }
}
```

**Node.js 對比：**
```javascript
// Node.js - mysql2
async function transferMoney(pool, fromUserID, toUserID, amount) {
  const connection = await pool.getConnection();

  try {
    // 開始事務
    await connection.beginTransaction();

    // 扣款
    await connection.execute(
      'UPDATE accounts SET balance = balance - ? WHERE user_id = ?',
      [amount, fromUserID]
    );

    // 入賬
    await connection.execute(
      'UPDATE accounts SET balance = balance + ? WHERE user_id = ?',
      [amount, toUserID]
    );

    // 記錄轉賬
    await connection.execute(
      'INSERT INTO transactions(from_user, to_user, amount) VALUES(?, ?, ?)',
      [fromUserID, toUserID, amount]
    );

    // 提交事務
    await connection.commit();
    console.log('事務提交成功');

  } catch (error) {
    // 回滾事務
    await connection.rollback();
    console.log('事務回滾');
    throw error;
  } finally {
    connection.release();
  }
}

// 使用
try {
  await transferMoney(pool, 1, 2, 100.00);
} catch (error) {
  console.error('轉賬失敗:', error);
}
```

### 5. 上下文 (Context) 支持

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "time"
)

func main() {
    db, _ := sql.Open("mysql",
        "root:password@tcp(localhost:3306)/mydb")
    defer db.Close()

    // 1. 帶超時的查詢
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    rows, err := db.QueryContext(ctx, "SELECT * FROM users")
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()

    // 2. 可取消的查詢
    ctx2, cancel2 := context.WithCancel(context.Background())

    go func() {
        time.Sleep(1 * time.Second)
        cancel2() // 1秒後取消查詢
    }()

    _, err = db.ExecContext(ctx2, "UPDATE users SET name = 'test'")
    if err == context.Canceled {
        fmt.Println("查詢被取消")
    }

    // 3. 事務中使用 Context
    ctx3, cancel3 := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel3()

    tx, err := db.BeginTx(ctx3, nil)
    if err != nil {
        log.Fatal(err)
    }
    defer tx.Rollback()

    _, err = tx.ExecContext(ctx3, "INSERT INTO users(name) VALUES(?)", "Alice")
    if err != nil {
        log.Fatal(err)
    }

    tx.Commit()
}
```

### 6. Null 值處理

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
)

type User struct {
    ID      int
    Name    string
    Email   sql.NullString  // 可能為 NULL
    Age     sql.NullInt64   // 可能為 NULL
    Bio     sql.NullString  // 可能為 NULL
}

func main() {
    db, _ := sql.Open("mysql",
        "root:password@tcp(localhost:3306)/mydb")
    defer db.Close()

    // 查詢包含 NULL 值的數據
    var user User
    err := db.QueryRow("SELECT id, name, email, age, bio FROM users WHERE id = ?", 1).
        Scan(&user.ID, &user.Name, &user.Email, &user.Age, &user.Bio)

    if err != nil {
        log.Fatal(err)
    }

    // 處理 NULL 值
    fmt.Printf("ID: %d\n", user.ID)
    fmt.Printf("Name: %s\n", user.Name)

    if user.Email.Valid {
        fmt.Printf("Email: %s\n", user.Email.String)
    } else {
        fmt.Println("Email: NULL")
    }

    if user.Age.Valid {
        fmt.Printf("Age: %d\n", user.Age.Int64)
    } else {
        fmt.Println("Age: NULL")
    }

    // 插入 NULL 值
    _, err = db.Exec(
        "INSERT INTO users(name, email, age) VALUES(?, ?, ?)",
        "Bob",
        sql.NullString{String: "", Valid: false}, // NULL
        sql.NullInt64{Int64: 25, Valid: true},    // 25
    )
}
```

**Node.js 對比：**
```javascript
// Node.js 中 NULL 處理更簡單
const [rows] = await pool.execute(
  'SELECT id, name, email, age, bio FROM users WHERE id = ?',
  [1]
);

const user = rows[0];
console.log('ID:', user.id);
console.log('Name:', user.name);
console.log('Email:', user.email || 'NULL');  // undefined 或 null 表示 NULL
console.log('Age:', user.age ?? 'NULL');

// 插入 NULL
await pool.execute(
  'INSERT INTO users(name, email, age) VALUES(?, ?, ?)',
  ['Bob', null, 25]  // 直接使用 null
);
```

## 完整示例：用戶管理系統

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "time"

    _ "github.com/go-sql-driver/mysql"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(name, email string, age int) (int64, error) {
    result, err := r.db.Exec(
        "INSERT INTO users(name, email, age, created_at) VALUES(?, ?, ?, ?)",
        name, email, age, time.Now(),
    )
    if err != nil {
        return 0, err
    }
    return result.LastInsertId()
}

func (r *UserRepository) GetByID(id int) (*User, error) {
    var u User
    err := r.db.QueryRow(
        "SELECT id, name, email, age FROM users WHERE id = ?",
        id,
    ).Scan(&u.ID, &u.Name, &u.Email, &u.Age)

    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    return &u, nil
}

func (r *UserRepository) List(limit, offset int) ([]User, error) {
    rows, err := r.db.Query(
        "SELECT id, name, email, age FROM users LIMIT ? OFFSET ?",
        limit, offset,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email, &u.Age); err != nil {
            return nil, err
        }
        users = append(users, u)
    }

    return users, rows.Err()
}

func (r *UserRepository) Update(id int, name, email string, age int) error {
    _, err := r.db.Exec(
        "UPDATE users SET name = ?, email = ?, age = ? WHERE id = ?",
        name, email, age, id,
    )
    return err
}

func (r *UserRepository) Delete(id int) error {
    _, err := r.db.Exec("DELETE FROM users WHERE id = ?", id)
    return err
}

func main() {
    db, err := sql.Open("mysql",
        "root:password@tcp(localhost:3306)/mydb?parseTime=true")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    repo := NewUserRepository(db)

    // 創建用戶
    id, err := repo.Create("Alice", "alice@example.com", 25)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("創建用戶，ID: %d\n", id)

    // 獲取用戶
    user, err := repo.GetByID(int(id))
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("用戶: %+v\n", user)

    // 更新用戶
    err = repo.Update(int(id), "Alice Smith", "alice.smith@example.com", 26)
    if err != nil {
        log.Fatal(err)
    }

    // 列出用戶
    users, err := repo.List(10, 0)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("用戶列表: %+v\n", users)
}
```

## 重點總結

### 關鍵差異

| 特性 | Node.js | Go database/sql |
|------|---------|-----------------|
| **API 一致性** | 每個驅動不同 | 統一接口 |
| **連接池** | 手動配置 | 內置自動管理 |
| **錯誤處理** | try-catch | 返回 error |
| **NULL 值** | null/undefined | sql.NullXxx 類型 |
| **事務** | async/await | defer 模式 |
| **超時控制** | 驅動特定 | Context 統一控制 |

### 最佳實踐

1. **總是關閉資源**
   ```go
   rows, err := db.Query(...)
   if err != nil {
       return err
   }
   defer rows.Close()  // 必須關閉
   ```

2. **檢查 rows.Err()**
   ```go
   for rows.Next() {
       // scan...
   }
   if err = rows.Err(); err != nil {  // 檢查迭代錯誤
       return err
   }
   ```

3. **使用 Context 控制超時**
   ```go
   ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
   defer cancel()
   rows, err := db.QueryContext(ctx, ...)
   ```

4. **正確配置連接池**
   ```go
   db.SetMaxOpenConns(25)
   db.SetMaxIdleConns(5)
   db.SetConnMaxLifetime(5 * time.Minute)
   ```

5. **使用預處理語句**
   - 防止 SQL 注入
   - 提高性能（多次執行）

## 練習題

1. **基礎練習**：創建一個博客文章管理系統，實現 CRUD 操作
2. **中級練習**：實現帶分頁和搜索功能的用戶列表
3. **高級練習**：實現一個完整的訂單系統，包含事務處理（創建訂單、扣減庫存、記錄日誌）
4. **實戰練習**：使用 Context 實現可取消的批量數據導入功能

## 下一步

在下一章中，我們將學習 GORM - Go 中最流行的 ORM 框架，它提供了更高級的數據庫操作抽象。
