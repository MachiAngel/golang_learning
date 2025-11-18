# Chapter 45: Redis 集成

## 概述

Redis 是一個高性能的內存數據庫，常用於緩存、會話存儲、消息隊列等場景。Go 通過 `go-redis` 庫提供了完整的 Redis 支持，功能類似於 Node.js 的 `ioredis`。

## Node.js vs Go 對比

### ioredis (Node.js)
```javascript
const Redis = require('ioredis');

// 連接
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  password: 'password',
  db: 0
});

// 基礎操作
await redis.set('key', 'value');
const value = await redis.get('key');

// Hash 操作
await redis.hset('user:1', 'name', 'Alice');
const name = await redis.hget('user:1', 'name');

// List 操作
await redis.lpush('queue', 'task1');
const task = await redis.rpop('queue');

// Pipeline
const pipeline = redis.pipeline();
pipeline.set('key1', 'value1');
pipeline.set('key2', 'value2');
await pipeline.exec();
```

### go-redis (Go)
```go
import (
    "github.com/redis/go-redis/v9"
)

// 連接
rdb := redis.NewClient(&redis.Options{
    Addr:     "localhost:6379",
    Password: "password",
    DB:       0,
})

// 基礎操作
err := rdb.Set(ctx, "key", "value", 0).Err()
value, err := rdb.Get(ctx, "key").Result()

// Hash 操作
err = rdb.HSet(ctx, "user:1", "name", "Alice").Err()
name, err := rdb.HGet(ctx, "user:1", "name").Result()

// List 操作
err = rdb.LPush(ctx, "queue", "task1").Err()
task, err := rdb.RPop(ctx, "queue").Result()

// Pipeline
pipe := rdb.Pipeline()
pipe.Set(ctx, "key1", "value1", 0)
pipe.Set(ctx, "key2", "value2", 0)
_, err = pipe.Exec(ctx)
```

## 安裝

```bash
go get github.com/redis/go-redis/v9
```

## 1. 連接與配置

### 基礎連接

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

func main() {
    ctx := context.Background()

    // 創建客戶端
    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",    // 無密碼
        DB:       0,     // 使用默認數據庫
    })
    defer rdb.Close()

    // 測試連接
    pong, err := rdb.Ping(ctx).Result()
    if err != nil {
        panic(err)
    }
    fmt.Println("連接成功:", pong)
}
```

### 高級配置

```go
package main

import (
    "context"
    "time"

    "github.com/redis/go-redis/v9"
)

func advancedConnection() *redis.Client {
    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "password",
        DB:       0,

        // 連接池配置
        PoolSize:     10,               // 連接池大小
        MinIdleConns: 5,                // 最小空閒連接
        MaxIdleConns: 10,               // 最大空閒連接
        PoolTimeout:  4 * time.Second,  // 連接池超時

        // 連接超時
        DialTimeout:  5 * time.Second,  // 連接超時
        ReadTimeout:  3 * time.Second,  // 讀超時
        WriteTimeout: 3 * time.Second,  // 寫超時

        // 空閒超時
        ConnMaxIdleTime: 5 * time.Minute,
        ConnMaxLifetime: 10 * time.Minute,

        // 重試
        MaxRetries:      3,
        MinRetryBackoff: 8 * time.Millisecond,
        MaxRetryBackoff: 512 * time.Millisecond,
    })

    return rdb
}
```

### 集群連接

```go
package main

import (
    "github.com/redis/go-redis/v9"
)

func clusterConnection() *redis.ClusterClient {
    rdb := redis.NewClusterClient(&redis.ClusterOptions{
        Addrs: []string{
            "localhost:7000",
            "localhost:7001",
            "localhost:7002",
        },
        Password: "password",

        // 其他配置與單機相同
        PoolSize:    10,
        DialTimeout: 5 * time.Second,
    })

    return rdb
}
```

**Node.js 對比：**
```javascript
const Redis = require('ioredis');

// 基礎連接
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  password: 'password',
  db: 0
});

// 高級配置
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  password: 'password',
  db: 0,
  maxRetriesPerRequest: 3,
  connectTimeout: 5000,
  retryStrategy: (times) => Math.min(times * 50, 2000)
});

// 集群
const cluster = new Redis.Cluster([
  { host: 'localhost', port: 7000 },
  { host: 'localhost', port: 7001 },
  { host: 'localhost', port: 7002 }
], {
  redisOptions: { password: 'password' }
});

// 測試連接
redis.ping().then(() => console.log('連接成功'));
```

## 2. 基礎數據類型操作

### String 操作

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

func stringOperations(rdb *redis.Client) {
    ctx := context.Background()

    // 1. SET/GET
    err := rdb.Set(ctx, "name", "Alice", 0).Err()
    if err != nil {
        panic(err)
    }

    val, err := rdb.Get(ctx, "name").Result()
    if err == redis.Nil {
        fmt.Println("key 不存在")
    } else if err != nil {
        panic(err)
    } else {
        fmt.Println("name:", val)
    }

    // 2. 帶過期時間
    err = rdb.Set(ctx, "session", "token123", 10*time.Minute).Err()

    // 3. SETEX（設置並指定過期時間）
    err = rdb.SetEx(ctx, "code", "123456", 5*time.Minute).Err()

    // 4. SETNX（不存在時設置）
    success, err := rdb.SetNX(ctx, "lock", "1", 30*time.Second).Result()
    if success {
        fmt.Println("獲取鎖成功")
    }

    // 5. MGET/MSET（批量操作）
    err = rdb.MSet(ctx, "key1", "value1", "key2", "value2").Err()
    vals, err := rdb.MGet(ctx, "key1", "key2").Result()
    fmt.Println("批量獲取:", vals)

    // 6. INCR/DECR（計數器）
    err = rdb.Set(ctx, "counter", 0, 0).Err()
    newVal, err := rdb.Incr(ctx, "counter").Result()
    fmt.Println("計數器:", newVal)

    incrBy, err := rdb.IncrBy(ctx, "counter", 5).Result()
    fmt.Println("增加5後:", incrBy)

    decrVal, err := rdb.Decr(ctx, "counter").Result()
    fmt.Println("減1後:", decrVal)

    // 7. APPEND
    length, err := rdb.Append(ctx, "message", "Hello").Result()
    length, err = rdb.Append(ctx, "message", " World").Result()
    msg, _ := rdb.Get(ctx, "message").Result()
    fmt.Println("拼接結果:", msg)

    // 8. GETSET（獲取舊值並設置新值）
    oldVal, err := rdb.GetSet(ctx, "name", "Bob").Result()
    fmt.Println("舊值:", oldVal)

    // 9. 過期時間操作
    err = rdb.Expire(ctx, "name", 1*time.Hour).Err()    // 設置過期時間
    ttl, err := rdb.TTL(ctx, "name").Result()           // 查詢剩餘時間
    fmt.Println("剩餘時間:", ttl)

    err = rdb.Persist(ctx, "name").Err()                // 移除過期時間
}
```

**Node.js 對比：**
```javascript
// String 操作
await redis.set('name', 'Alice');
const val = await redis.get('name');

// 帶過期時間（秒）
await redis.setex('session', 600, 'token123');

// SETNX
const success = await redis.setnx('lock', '1');

// 批量操作
await redis.mset('key1', 'value1', 'key2', 'value2');
const vals = await redis.mget('key1', 'key2');

// 計數器
await redis.incr('counter');
await redis.incrby('counter', 5);
await redis.decr('counter');

// TTL
await redis.expire('name', 3600);
const ttl = await redis.ttl('name');
```

### Hash 操作

```go
package main

import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

func hashOperations(rdb *redis.Client) {
    ctx := context.Background()

    // 1. HSET/HGET
    err := rdb.HSet(ctx, "user:1", "name", "Alice", "age", 25).Err()
    name, err := rdb.HGet(ctx, "user:1", "name").Result()
    fmt.Println("姓名:", name)

    // 2. HMGET（批量獲取）
    vals, err := rdb.HMGet(ctx, "user:1", "name", "age").Result()
    fmt.Println("批量獲取:", vals)

    // 3. HGETALL（獲取所有字段）
    all, err := rdb.HGetAll(ctx, "user:1").Result()
    fmt.Println("所有字段:", all)

    // 4. HDEL（刪除字段）
    err = rdb.HDel(ctx, "user:1", "age").Err()

    // 5. HEXISTS（檢查字段是否存在）
    exists, err := rdb.HExists(ctx, "user:1", "name").Result()
    fmt.Println("字段存在:", exists)

    // 6. HKEYS/HVALS（獲取所有鍵/值）
    keys, err := rdb.HKeys(ctx, "user:1").Result()
    vals2, err := rdb.HVals(ctx, "user:1").Result()
    fmt.Println("所有鍵:", keys)
    fmt.Println("所有值:", vals2)

    // 7. HLEN（字段數量）
    count, err := rdb.HLen(ctx, "user:1").Result()
    fmt.Println("字段數量:", count)

    // 8. HINCRBY（增加數字）
    newAge, err := rdb.HIncrBy(ctx, "user:1", "age", 1).Result()
    fmt.Println("新年齡:", newAge)

    // 9. HSETNX（不存在時設置）
    success, err := rdb.HSetNX(ctx, "user:1", "email", "alice@example.com").Result()
}
```

**Node.js 對比：**
```javascript
// Hash 操作
await redis.hset('user:1', 'name', 'Alice', 'age', 25);
const name = await redis.hget('user:1', 'name');

// 批量操作
const vals = await redis.hmget('user:1', 'name', 'age');
const all = await redis.hgetall('user:1');

// 其他操作
await redis.hdel('user:1', 'age');
const exists = await redis.hexists('user:1', 'name');
const keys = await redis.hkeys('user:1');
const count = await redis.hlen('user:1');
await redis.hincrby('user:1', 'age', 1);
```

### List 操作

```go
package main

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

func listOperations(rdb *redis.Client) {
    ctx := context.Background()

    // 1. LPUSH/RPUSH（左插入/右插入）
    err := rdb.LPush(ctx, "queue", "task1", "task2").Err()
    err = rdb.RPush(ctx, "queue", "task3").Err()

    // 2. LPOP/RPOP（左彈出/右彈出）
    val, err := rdb.LPop(ctx, "queue").Result()
    fmt.Println("彈出:", val)

    // 3. BLPOP/BRPOP（阻塞彈出）
    result, err := rdb.BLPop(ctx, 5*time.Second, "queue").Result()
    if err == redis.Nil {
        fmt.Println("超時")
    } else {
        fmt.Println("阻塞彈出:", result)
    }

    // 4. LRANGE（獲取範圍）
    items, err := rdb.LRange(ctx, "queue", 0, -1).Result()
    fmt.Println("所有元素:", items)

    // 5. LLEN（長度）
    length, err := rdb.LLen(ctx, "queue").Result()
    fmt.Println("列表長度:", length)

    // 6. LINDEX（獲取指定位置）
    item, err := rdb.LIndex(ctx, "queue", 0).Result()
    fmt.Println("第一個元素:", item)

    // 7. LSET（設置指定位置）
    err = rdb.LSet(ctx, "queue", 0, "newTask").Err()

    // 8. LREM（刪除元素）
    removed, err := rdb.LRem(ctx, "queue", 1, "task1").Result()
    fmt.Println("刪除數量:", removed)

    // 9. LTRIM（修剪列表）
    err = rdb.LTrim(ctx, "queue", 0, 9).Err() // 只保留前10個
}
```

**Node.js 對比：**
```javascript
// List 操作
await redis.lpush('queue', 'task1', 'task2');
await redis.rpush('queue', 'task3');

const val = await redis.lpop('queue');
const result = await redis.blpop('queue', 5);

const items = await redis.lrange('queue', 0, -1);
const length = await redis.llen('queue');
const item = await redis.lindex('queue', 0);

await redis.lset('queue', 0, 'newTask');
await redis.lrem('queue', 1, 'task1');
await redis.ltrim('queue', 0, 9);
```

### Set 操作

```go
package main

import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

func setOperations(rdb *redis.Client) {
    ctx := context.Background()

    // 1. SADD（添加成員）
    err := rdb.SAdd(ctx, "tags", "go", "redis", "database").Err()

    // 2. SMEMBERS（獲取所有成員）
    members, err := rdb.SMembers(ctx, "tags").Result()
    fmt.Println("所有成員:", members)

    // 3. SISMEMBER（檢查成員是否存在）
    exists, err := rdb.SIsMember(ctx, "tags", "go").Result()
    fmt.Println("成員存在:", exists)

    // 4. SCARD（集合大小）
    size, err := rdb.SCard(ctx, "tags").Result()
    fmt.Println("集合大小:", size)

    // 5. SREM（刪除成員）
    err = rdb.SRem(ctx, "tags", "redis").Err()

    // 6. SPOP（隨機彈出）
    member, err := rdb.SPop(ctx, "tags").Result()
    fmt.Println("隨機彈出:", member)

    // 7. SRANDMEMBER（隨機獲取，不刪除）
    randMember, err := rdb.SRandMember(ctx, "tags").Result()
    fmt.Println("隨機獲取:", randMember)

    // 8. 集合運算
    rdb.SAdd(ctx, "set1", "a", "b", "c")
    rdb.SAdd(ctx, "set2", "b", "c", "d")

    // 並集
    union, err := rdb.SUnion(ctx, "set1", "set2").Result()
    fmt.Println("並集:", union)

    // 交集
    inter, err := rdb.SInter(ctx, "set1", "set2").Result()
    fmt.Println("交集:", inter)

    // 差集
    diff, err := rdb.SDiff(ctx, "set1", "set2").Result()
    fmt.Println("差集:", diff)

    // 存儲結果
    err = rdb.SUnionStore(ctx, "set3", "set1", "set2").Err()
}
```

### Sorted Set 操作

```go
package main

import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

func zsetOperations(rdb *redis.Client) {
    ctx := context.Background()

    // 1. ZADD（添加成員）
    err := rdb.ZAdd(ctx, "leaderboard", redis.Z{Score: 100, Member: "Alice"}).Err()
    err = rdb.ZAdd(ctx, "leaderboard",
        redis.Z{Score: 90, Member: "Bob"},
        redis.Z{Score: 95, Member: "Charlie"},
    ).Err()

    // 2. ZRANGE（按分數升序）
    members, err := rdb.ZRange(ctx, "leaderboard", 0, -1).Result()
    fmt.Println("升序:", members)

    // 3. ZREVRANGE（按分數降序）
    members, err = rdb.ZRevRange(ctx, "leaderboard", 0, -1).Result()
    fmt.Println("降序:", members)

    // 4. ZRANGE WITH SCORES（帶分數）
    membersWithScores, err := rdb.ZRangeWithScores(ctx, "leaderboard", 0, -1).Result()
    for _, z := range membersWithScores {
        fmt.Printf("%s: %.0f\n", z.Member, z.Score)
    }

    // 5. ZRANK（獲取排名，從0開始）
    rank, err := rdb.ZRank(ctx, "leaderboard", "Alice").Result()
    fmt.Println("Alice 排名:", rank)

    // 6. ZSCORE（獲取分數）
    score, err := rdb.ZScore(ctx, "leaderboard", "Alice").Result()
    fmt.Println("Alice 分數:", score)

    // 7. ZINCRBY（增加分數）
    newScore, err := rdb.ZIncrBy(ctx, "leaderboard", 10, "Alice").Result()
    fmt.Println("新分數:", newScore)

    // 8. ZCARD（成員數量）
    count, err := rdb.ZCard(ctx, "leaderboard").Result()
    fmt.Println("成員數量:", count)

    // 9. ZCOUNT（分數範圍內的成員數）
    count, err = rdb.ZCount(ctx, "leaderboard", "90", "100").Result()
    fmt.Println("90-100分的人數:", count)

    // 10. ZRANGEBYSCORE（按分數範圍查詢）
    members, err = rdb.ZRangeByScore(ctx, "leaderboard", &redis.ZRangeBy{
        Min: "90",
        Max: "100",
    }).Result()
    fmt.Println("90-100分:", members)

    // 11. ZREM（刪除成員）
    err = rdb.ZRem(ctx, "leaderboard", "Bob").Err()

    // 12. ZREMRANGEBYSCORE（按分數範圍刪除）
    removed, err := rdb.ZRemRangeByScore(ctx, "leaderboard", "0", "50").Result()
    fmt.Println("刪除數量:", removed)
}
```

**Node.js 對比：**
```javascript
// Sorted Set 操作
await redis.zadd('leaderboard', 100, 'Alice', 90, 'Bob', 95, 'Charlie');

const members = await redis.zrange('leaderboard', 0, -1);
const descMembers = await redis.zrevrange('leaderboard', 0, -1);

// 帶分數
const withScores = await redis.zrange('leaderboard', 0, -1, 'WITHSCORES');

const rank = await redis.zrank('leaderboard', 'Alice');
const score = await redis.zscore('leaderboard', 'Alice');

await redis.zincrby('leaderboard', 10, 'Alice');

const count = await redis.zcard('leaderboard');
const rangeCount = await redis.zcount('leaderboard', 90, 100);

const rangeMembers = await redis.zrangebyscore('leaderboard', 90, 100);

await redis.zrem('leaderboard', 'Bob');
```

## 3. 高級功能

### Pipeline（管道）

```go
package main

import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

func pipelineExample(rdb *redis.Client) {
    ctx := context.Background()

    // 創建 Pipeline
    pipe := rdb.Pipeline()

    // 添加命令（不會立即執行）
    incr := pipe.Incr(ctx, "counter")
    pipe.Set(ctx, "key1", "value1", 0)
    pipe.Get(ctx, "key1")

    // 執行所有命令
    cmds, err := pipe.Exec(ctx)
    if err != nil {
        panic(err)
    }

    fmt.Println("執行了", len(cmds), "個命令")
    fmt.Println("counter 值:", incr.Val())

    // 使用 Pipelined 簡化寫法
    var incr2 *redis.IntCmd
    _, err = rdb.Pipelined(ctx, func(pipe redis.Pipeliner) error {
        incr2 = pipe.Incr(ctx, "counter")
        pipe.Set(ctx, "key2", "value2", 0)
        return nil
    })

    fmt.Println("counter 新值:", incr2.Val())
}
```

**Node.js 對比：**
```javascript
// Pipeline
const pipeline = redis.pipeline();
pipeline.incr('counter');
pipeline.set('key1', 'value1');
pipeline.get('key1');

const results = await pipeline.exec();
console.log('執行了', results.length, '個命令');
console.log('counter 值:', results[0][1]);
```

### Transaction（事務）

```go
package main

import (
    "context"
    "errors"

    "github.com/redis/go-redis/v9"
)

func transactionExample(rdb *redis.Client) {
    ctx := context.Background()

    // 使用 TxPipelined
    err := rdb.Watch(ctx, func(tx *redis.Tx) error {
        // 在事務中執行命令
        _, err := tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
            pipe.Set(ctx, "key1", "value1", 0)
            pipe.Set(ctx, "key2", "value2", 0)
            return nil
        })
        return err
    }, "key1", "key2") // 監視這些鍵

    // 樂觀鎖示例
    const maxRetries = 100

    increment := func(key string) error {
        return rdb.Watch(ctx, func(tx *redis.Tx) error {
            // 獲取當前值
            n, err := tx.Get(ctx, key).Int()
            if err != nil && err != redis.Nil {
                return err
            }

            // 在事務中更新
            _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
                pipe.Set(ctx, key, n+1, 0)
                return nil
            })
            return err
        }, key)
    }

    // 重試邏輯
    for i := 0; i < maxRetries; i++ {
        err := increment("counter")
        if err == nil {
            break
        }
        if err == redis.TxFailedErr {
            continue // 重試
        }
        panic(err)
    }
}
```

**Node.js 對比：**
```javascript
// Transaction
const multi = redis.multi();
multi.set('key1', 'value1');
multi.set('key2', 'value2');
await multi.exec();

// 樂觀鎖
const result = await redis.watch('counter', async () => {
  const val = await redis.get('counter');
  const multi = redis.multi();
  multi.set('counter', parseInt(val) + 1);
  return multi.exec();
});
```

### Pub/Sub（發布訂閱）

```go
package main

import (
    "context"
    "fmt"

    "github.com/redis/go-redis/v9"
)

func pubsubExample(rdb *redis.Client) {
    ctx := context.Background()

    // 訂閱者
    go func() {
        pubsub := rdb.Subscribe(ctx, "channel1", "channel2")
        defer pubsub.Close()

        // 等待訂閱確認
        _, err := pubsub.Receive(ctx)
        if err != nil {
            panic(err)
        }

        // 接收消息
        ch := pubsub.Channel()
        for msg := range ch {
            fmt.Printf("收到消息 [%s]: %s\n", msg.Channel, msg.Payload)
        }
    }()

    // 發布者
    err := rdb.Publish(ctx, "channel1", "Hello World").Err()
    if err != nil {
        panic(err)
    }

    // 模式訂閱
    go func() {
        pubsub := rdb.PSubscribe(ctx, "news.*")
        defer pubsub.Close()

        ch := pubsub.Channel()
        for msg := range ch {
            fmt.Printf("模式匹配 [%s]: %s\n", msg.Channel, msg.Payload)
        }
    }()

    // 發布到匹配的頻道
    rdb.Publish(ctx, "news.sports", "Sports news")
    rdb.Publish(ctx, "news.tech", "Tech news")
}
```

**Node.js 對比：**
```javascript
// 訂閱者
const subscriber = redis.duplicate();
await subscriber.subscribe('channel1', 'channel2');

subscriber.on('message', (channel, message) => {
  console.log(`收到消息 [${channel}]: ${message}`);
});

// 發布者
await redis.publish('channel1', 'Hello World');

// 模式訂閱
await subscriber.psubscribe('news.*');
subscriber.on('pmessage', (pattern, channel, message) => {
  console.log(`模式匹配 [${channel}]: ${message}`);
});
```

### Lua 腳本

```go
package main

import (
    "context"

    "github.com/redis/go-redis/v9"
)

func luaScriptExample(rdb *redis.Client) {
    ctx := context.Background()

    // 定義 Lua 腳本
    script := redis.NewScript(`
        local current = redis.call('GET', KEYS[1])
        if current == false then
            current = 0
        end
        current = tonumber(current) + tonumber(ARGV[1])
        redis.call('SET', KEYS[1], current)
        return current
    `)

    // 執行腳本
    result, err := script.Run(ctx, rdb, []string{"counter"}, 10).Result()
    if err != nil {
        panic(err)
    }
    fmt.Println("腳本執行結果:", result)

    // 限流腳本示例
    rateLimitScript := redis.NewScript(`
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local current = redis.call('INCR', key)

        if current == 1 then
            redis.call('EXPIRE', key, window)
        end

        if current > limit then
            return 0
        else
            return 1
        end
    `)

    // 檢查限流
    allowed, err := rateLimitScript.Run(
        ctx,
        rdb,
        []string{"rate_limit:user:1"},
        100,  // 限制次數
        60,   // 時間窗口（秒）
    ).Int()

    if allowed == 1 {
        fmt.Println("請求允許")
    } else {
        fmt.Println("請求被限流")
    }
}
```

**Node.js 對比：**
```javascript
// Lua 腳本
const script = `
  local current = redis.call('GET', KEYS[1])
  if current == false then current = 0 end
  current = tonumber(current) + tonumber(ARGV[1])
  redis.call('SET', KEYS[1], current)
  return current
`;

const result = await redis.eval(script, 1, 'counter', 10);
console.log('腳本執行結果:', result);

// 或使用 defineCommand
redis.defineCommand('incrBy', {
  numberOfKeys: 1,
  lua: script
});

const result = await redis.incrBy('counter', 10);
```

## 4. 實戰應用

### 緩存系統

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type CacheService struct {
    rdb *redis.Client
}

func NewCacheService(rdb *redis.Client) *CacheService {
    return &CacheService{rdb: rdb}
}

// 設置緩存
func (s *CacheService) Set(ctx context.Context, key string, value interface{}, ttl time.Duration) error {
    data, err := json.Marshal(value)
    if err != nil {
        return err
    }
    return s.rdb.Set(ctx, key, data, ttl).Err()
}

// 獲取緩存
func (s *CacheService) Get(ctx context.Context, key string, dest interface{}) error {
    data, err := s.rdb.Get(ctx, key).Bytes()
    if err != nil {
        return err
    }
    return json.Unmarshal(data, dest)
}

// 刪除緩存
func (s *CacheService) Delete(ctx context.Context, keys ...string) error {
    return s.rdb.Del(ctx, keys...).Err()
}

// Cache-Aside 模式
func (s *CacheService) GetOrSet(ctx context.Context, key string, ttl time.Duration, loader func() (interface{}, error), dest interface{}) error {
    // 嘗試從緩存獲取
    err := s.Get(ctx, key, dest)
    if err == nil {
        return nil // 緩存命中
    }
    if err != redis.Nil {
        return err
    }

    // 緩存未命中，從數據源加載
    data, err := loader()
    if err != nil {
        return err
    }

    // 設置緩存
    if err := s.Set(ctx, key, data, ttl); err != nil {
        return err
    }

    // 返回數據
    jsonData, _ := json.Marshal(data)
    return json.Unmarshal(jsonData, dest)
}

// 使用示例
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func cacheExample() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    cache := NewCacheService(rdb)

    // 設置緩存
    user := User{ID: 1, Name: "Alice"}
    cache.Set(ctx, "user:1", user, 10*time.Minute)

    // 獲取緩存
    var cachedUser User
    err := cache.Get(ctx, "user:1", &cachedUser)
    if err == redis.Nil {
        fmt.Println("緩存不存在")
    } else if err != nil {
        panic(err)
    } else {
        fmt.Printf("從緩存獲取: %+v\n", cachedUser)
    }

    // Cache-Aside
    var user2 User
    err = cache.GetOrSet(ctx, "user:2", 10*time.Minute, func() (interface{}, error) {
        // 從數據庫加載
        return User{ID: 2, Name: "Bob"}, nil
    }, &user2)
    fmt.Printf("用戶: %+v\n", user2)
}
```

### 會話管理

```go
package main

import (
    "context"
    "encoding/json"
    "time"

    "github.com/google/uuid"
    "github.com/redis/go-redis/v9"
)

type SessionService struct {
    rdb *redis.Client
    ttl time.Duration
}

func NewSessionService(rdb *redis.Client, ttl time.Duration) *SessionService {
    return &SessionService{rdb: rdb, ttl: ttl}
}

// 創建會話
func (s *SessionService) Create(ctx context.Context, userID int, data map[string]interface{}) (string, error) {
    sessionID := uuid.New().String()
    key := "session:" + sessionID

    sessionData := map[string]interface{}{
        "userID":    userID,
        "createdAt": time.Now(),
        "data":      data,
    }

    jsonData, err := json.Marshal(sessionData)
    if err != nil {
        return "", err
    }

    err = s.rdb.Set(ctx, key, jsonData, s.ttl).Err()
    return sessionID, err
}

// 獲取會話
func (s *SessionService) Get(ctx context.Context, sessionID string) (map[string]interface{}, error) {
    key := "session:" + sessionID

    data, err := s.rdb.Get(ctx, key).Bytes()
    if err != nil {
        return nil, err
    }

    var session map[string]interface{}
    err = json.Unmarshal(data, &session)
    return session, err
}

// 更新會話
func (s *SessionService) Update(ctx context.Context, sessionID string, data map[string]interface{}) error {
    session, err := s.Get(ctx, sessionID)
    if err != nil {
        return err
    }

    session["data"] = data

    jsonData, err := json.Marshal(session)
    if err != nil {
        return err
    }

    key := "session:" + sessionID
    return s.rdb.Set(ctx, key, jsonData, s.ttl).Err()
}

// 刷新會話
func (s *SessionService) Refresh(ctx context.Context, sessionID string) error {
    key := "session:" + sessionID
    return s.rdb.Expire(ctx, key, s.ttl).Err()
}

// 銷毀會話
func (s *SessionService) Destroy(ctx context.Context, sessionID string) error {
    key := "session:" + sessionID
    return s.rdb.Del(ctx, key).Err()
}
```

### 分布式鎖

```go
package main

import (
    "context"
    "errors"
    "time"

    "github.com/google/uuid"
    "github.com/redis/go-redis/v9"
)

type DistributedLock struct {
    rdb   *redis.Client
    key   string
    value string
    ttl   time.Duration
}

func NewDistributedLock(rdb *redis.Client, key string, ttl time.Duration) *DistributedLock {
    return &DistributedLock{
        rdb:   rdb,
        key:   "lock:" + key,
        value: uuid.New().String(),
        ttl:   ttl,
    }
}

// 獲取鎖
func (l *DistributedLock) Acquire(ctx context.Context) (bool, error) {
    success, err := l.rdb.SetNX(ctx, l.key, l.value, l.ttl).Result()
    return success, err
}

// 釋放鎖（使用 Lua 腳本保證原子性）
func (l *DistributedLock) Release(ctx context.Context) error {
    script := redis.NewScript(`
        if redis.call("GET", KEYS[1]) == ARGV[1] then
            return redis.call("DEL", KEYS[1])
        else
            return 0
        end
    `)

    result, err := script.Run(ctx, l.rdb, []string{l.key}, l.value).Int()
    if err != nil {
        return err
    }
    if result == 0 {
        return errors.New("鎖已被其他進程持有")
    }
    return nil
}

// 使用示例
func distributedLockExample() {
    ctx := context.Background()
    rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

    lock := NewDistributedLock(rdb, "resource:1", 30*time.Second)

    // 嘗試獲取鎖
    acquired, err := lock.Acquire(ctx)
    if err != nil {
        panic(err)
    }

    if !acquired {
        fmt.Println("無法獲取鎖")
        return
    }

    defer lock.Release(ctx)

    // 執行業務邏輯
    fmt.Println("執行關鍵操作...")
    time.Sleep(2 * time.Second)
    fmt.Println("操作完成")
}
```

### 限流器

```go
package main

import (
    "context"
    "time"

    "github.com/redis/go-redis/v9"
)

type RateLimiter struct {
    rdb *redis.Client
}

func NewRateLimiter(rdb *redis.Client) *RateLimiter {
    return &RateLimiter{rdb: rdb}
}

// 滑動窗口限流
func (r *RateLimiter) Allow(ctx context.Context, key string, limit int, window time.Duration) (bool, error) {
    script := redis.NewScript(`
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        -- 刪除過期數據
        redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

        -- 獲取當前計數
        local current = redis.call('ZCARD', key)

        if current < limit then
            redis.call('ZADD', key, now, now)
            redis.call('EXPIRE', key, window)
            return 1
        else
            return 0
        end
    `)

    now := time.Now().Unix()
    result, err := script.Run(
        ctx,
        r.rdb,
        []string{"rate_limit:" + key},
        limit,
        int(window.Seconds()),
        now,
    ).Int()

    return result == 1, err
}

// 令牌桶限流
func (r *RateLimiter) TokenBucket(ctx context.Context, key string, capacity, refillRate int, window time.Duration) (bool, error) {
    script := redis.NewScript(`
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1])
        local last_refill = tonumber(bucket[2])

        if tokens == nil then
            tokens = capacity
            last_refill = now
        else
            local elapsed = now - last_refill
            local refilled = math.floor(elapsed * refill_rate)
            tokens = math.min(capacity, tokens + refilled)
            last_refill = now
        end

        if tokens >= 1 then
            tokens = tokens - 1
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', last_refill)
            redis.call('EXPIRE', key, 3600)
            return 1
        else
            return 0
        end
    `)

    result, err := script.Run(
        ctx,
        r.rdb,
        []string{"token_bucket:" + key},
        capacity,
        refillRate,
        time.Now().Unix(),
    ).Int()

    return result == 1, err
}
```

## 重點總結

### go-redis vs ioredis

| 特性 | go-redis | ioredis |
|------|----------|---------|
| **錯誤處理** | 返回 error | Promise reject |
| **Pipeline** | Pipeline() | pipeline() |
| **事務** | TxPipelined | multi() |
| **Pub/Sub** | Subscribe | subscribe |
| **集群** | ClusterClient | Cluster |

### 最佳實踐

1. **使用連接池**（默認已啟用）
2. **使用 Context 控制超時**
3. **Pipeline 批量操作**
4. **Lua 腳本保證原子性**
5. **合理設置過期時間**

## 練習題

1. **基礎**：實現一個簡單的計數器 API
2. **中級**：實現一個帶過期時間的緩存系統
3. **高級**：實現一個分布式限流中間件
4. **實戰**：構建一個完整的會話管理系統，包含創建、更新、刷新、銷毀功能

## 下一步

下一章將學習 Go 的測試框架，包括單元測試、基準測試和表格驅動測試。
