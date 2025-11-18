# Chapter 47: 基準測試 (Benchmarking)

## 概述

Go 的 `testing` 包內置了基準測試功能，用於測量代碼性能。與 Node.js 的 `benchmark.js` 不同，Go 的基準測試更加簡潔且集成在標準測試框架中。

## Node.js vs Go 對比

### benchmark.js (Node.js)
```javascript
const Benchmark = require('benchmark');
const suite = new Benchmark.Suite;

suite.add('Array#push', function() {
  const arr = [];
  for (let i = 0; i < 1000; i++) {
    arr.push(i);
  }
})
.add('Array#concat', function() {
  let arr = [];
  for (let i = 0; i < 1000; i++) {
    arr = arr.concat([i]);
  }
})
.on('complete', function() {
  console.log('Fastest is ' + this.filter('fastest').map('name'));
})
.run({ 'async': true });

// 輸出：
// Array#push x 123,456 ops/sec ±1.23%
// Array#concat x 12,345 ops/sec ±2.34%
// Fastest is Array#push
```

### testing.B (Go)
```go
package main

import "testing"

func BenchmarkArrayPush(b *testing.B) {
    for i := 0; i < b.N; i++ {
        arr := make([]int, 0)
        for j := 0; j < 1000; j++ {
            arr = append(arr, j)
        }
    }
}

func BenchmarkArrayConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        arr := make([]int, 0)
        for j := 0; j < 1000; j++ {
            arr = append(arr, []int{j}...)
        }
    }
}

// 運行：go test -bench=.
// 輸出：
// BenchmarkArrayPush-8     50000    25000 ns/op
// BenchmarkArrayConcat-8    5000   250000 ns/op
```

## 1. 基礎基準測試

### 基準測試規則

```go
package benchmark

import "testing"

// 1. 函數必須以 Benchmark 開頭
// 2. 接受一個參數 *testing.B
// 3. 必須運行 b.N 次循環

func BenchmarkSimple(b *testing.B) {
    // b.N 由測試框架自動調整，以獲得穩定的結果
    for i := 0; i < b.N; i++ {
        // 要測試的代碼
        _ = 2 + 2
    }
}

func BenchmarkStringConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "a"
        }
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("a")
        }
        _ = sb.String()
    }
}
```

### 運行基準測試

```bash
# 運行所有基準測試
go test -bench=.

# 運行特定基準測試
go test -bench=BenchmarkStringConcat

# 運行匹配模式的基準測試
go test -bench=String

# 指定運行時間（默認 1 秒）
go test -bench=. -benchtime=10s

# 指定迭代次數
go test -bench=. -benchtime=1000x

# 顯示內存分配
go test -bench=. -benchmem

# 輸出詳細信息
go test -bench=. -v

# 只運行基準測試，不運行單元測試
go test -run=^$ -bench=.
```

**輸出解讀：**
```
BenchmarkStringConcat-8      100000    15000 ns/op    2048 B/op    100 allocs/op
```
- `BenchmarkStringConcat-8`: 測試名稱-GOMAXPROCS
- `100000`: 運行了 10 萬次
- `15000 ns/op`: 每次操作耗時 15 微秒
- `2048 B/op`: 每次操作分配 2KB 內存（需要 -benchmem）
- `100 allocs/op`: 每次操作進行了 100 次內存分配

**Node.js 對比：**
```bash
# benchmark.js
node benchmark.js

# 輸出：
# StringConcat x 10,000 ops/sec ±1.23%
# StringBuilder x 100,000 ops/sec ±2.34%
```

## 2. 高級基準測試技巧

### 重置計時器

```go
package benchmark

import "testing"

func BenchmarkWithSetup(b *testing.B) {
    // Setup（不計入測試時間）
    data := make([]int, 1000000)
    for i := range data {
        data[i] = i
    }

    // 重置計時器
    b.ResetTimer()

    // 實際測試
    for i := 0; i < b.N; i++ {
        sum := 0
        for _, v := range data {
            sum += v
        }
    }
}

func BenchmarkWithStopStart(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // 暫停計時器
        b.StopTimer()

        // Setup（不計時）
        data := generateTestData()

        // 恢復計時器
        b.StartTimer()

        // 實際測試
        process(data)
    }
}
```

### 並行基準測試

```go
package benchmark

import (
    "sync"
    "testing"
)

func BenchmarkMapRead(b *testing.B) {
    m := make(map[int]int)
    for i := 0; i < 1000; i++ {
        m[i] = i
    }

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            _ = m[42]
        }
    })
}

func BenchmarkSyncMap(b *testing.B) {
    var m sync.Map
    for i := 0; i < 1000; i++ {
        m.Store(i, i)
    }

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            _, _ = m.Load(42)
        }
    })
}

// 運行：go test -bench=. -cpu=1,2,4,8
```

### 子基準測試

```go
package benchmark

import (
    "testing"
)

func BenchmarkSort(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("Size%d", size), func(b *testing.B) {
            data := generateData(size)
            b.ResetTimer()

            for i := 0; i < b.N; i++ {
                sort.Ints(data)
            }
        })
    }
}

// 輸出：
// BenchmarkSort/Size10-8         1000000    1000 ns/op
// BenchmarkSort/Size100-8         100000   15000 ns/op
// BenchmarkSort/Size1000-8         10000  150000 ns/op
// BenchmarkSort/Size10000-8         1000 1500000 ns/op
```

## 3. 內存分配分析

### 測量內存分配

```go
package benchmark

import (
    "strings"
    "testing"
)

func BenchmarkStringConcat(b *testing.B) {
    b.ReportAllocs() // 報告內存分配

    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 10; j++ {
            s += "hello"
        }
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 10; j++ {
            sb.WriteString("hello")
        }
        _ = sb.String()
    }
}

func BenchmarkStringJoin(b *testing.B) {
    b.ReportAllocs()

    for i := 0; i < b.N; i++ {
        parts := make([]string, 10)
        for j := 0; j < 10; j++ {
            parts[j] = "hello"
        }
        _ = strings.Join(parts, "")
    }
}

// 運行：go test -bench=. -benchmem
// 輸出：
// BenchmarkStringConcat-8      100000   15000 ns/op   1024 B/op    10 allocs/op
// BenchmarkStringBuilder-8    1000000    1500 ns/op     64 B/op     1 allocs/op
// BenchmarkStringJoin-8        500000    3000 ns/op    256 B/op     2 allocs/op
```

### 設置內存分配

```go
package benchmark

import "testing"

func BenchmarkWithBytes(b *testing.B) {
    // 手動設置每次操作的字節數
    b.SetBytes(1024)

    for i := 0; i < b.N; i++ {
        data := make([]byte, 1024)
        _ = data
    }
}

// 輸出會包含吞吐量信息
// BenchmarkWithBytes-8    1000000    1000 ns/op    1024.00 MB/s
```

## 4. 比較基準測試

### 使用 benchcmp 比較結果

```bash
# 運行基準測試並保存結果
go test -bench=. -benchmem > old.txt

# 修改代碼後重新運行
go test -bench=. -benchmem > new.txt

# 安裝 benchcmp
go install golang.org/x/tools/cmd/benchcmp@latest

# 比較結果
benchcmp old.txt new.txt

# 輸出：
# benchmark                old ns/op    new ns/op    delta
# BenchmarkStringConcat    15000        1500        -90.00%
#
# benchmark                old allocs   new allocs   delta
# BenchmarkStringConcat    10           1           -90.00%
```

### 使用 benchstat 進行統計分析

```bash
# 安裝 benchstat
go install golang.org/x/perf/cmd/benchstat@latest

# 多次運行基準測試
go test -bench=. -count=10 > benchmark.txt

# 統計分析
benchstat benchmark.txt

# 輸出：
# name                time/op
# StringConcat-8      15.0µs ± 2%
# StringBuilder-8     1.50µs ± 1%
#
# name                alloc/op
# StringConcat-8      1.02kB ± 0%
# StringBuilder-8     64.0B ± 0%
```

## 5. 完整示例：數據結構性能對比

```go
// benchmark_test.go
package datastructure

import (
    "container/list"
    "testing"
)

// 測試不同數據結構的性能

// Slice
func BenchmarkSliceAppend(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := make([]int, 0)
        for j := 0; j < 1000; j++ {
            s = append(s, j)
        }
    }
}

func BenchmarkSlicePrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := make([]int, 0, 1000)
        for j := 0; j < 1000; j++ {
            s = append(s, j)
        }
    }
}

// Map
func BenchmarkMapInsert(b *testing.B) {
    for i := 0; i < b.N; i++ {
        m := make(map[int]int)
        for j := 0; j < 1000; j++ {
            m[j] = j
        }
    }
}

func BenchmarkMapPrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        m := make(map[int]int, 1000)
        for j := 0; j < 1000; j++ {
            m[j] = j
        }
    }
}

// Linked List
func BenchmarkLinkedList(b *testing.B) {
    for i := 0; i < b.N; i++ {
        l := list.New()
        for j := 0; j < 1000; j++ {
            l.PushBack(j)
        }
    }
}

// 查找性能
func BenchmarkSliceSearch(b *testing.B) {
    s := make([]int, 10000)
    for i := range s {
        s[i] = i
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        for j := 0; j < len(s); j++ {
            if s[j] == 5000 {
                break
            }
        }
    }
}

func BenchmarkMapSearch(b *testing.B) {
    m := make(map[int]int, 10000)
    for i := 0; i < 10000; i++ {
        m[i] = i
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = m[5000]
    }
}

// 字符串操作對比
func BenchmarkStringOps(b *testing.B) {
    tests := []struct {
        name string
        fn   func(b *testing.B)
    }{
        {"Concat", benchmarkStringConcat},
        {"Builder", benchmarkStringBuilder},
        {"Join", benchmarkStringJoin},
        {"Format", benchmarkStringFormat},
    }

    for _, tt := range tests {
        b.Run(tt.name, tt.fn)
    }
}

func benchmarkStringConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "hello"
        }
    }
}

func benchmarkStringBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("hello")
        }
        _ = sb.String()
    }
}

func benchmarkStringJoin(b *testing.B) {
    for i := 0; i < b.N; i++ {
        parts := make([]string, 100)
        for j := 0; j < 100; j++ {
            parts[j] = "hello"
        }
        _ = strings.Join(parts, "")
    }
}

func benchmarkStringFormat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var parts []string
        for j := 0; j < 100; j++ {
            parts = append(parts, "hello")
        }
        _ = fmt.Sprint(parts)
    }
}
```

```bash
# 運行並查看內存分配
go test -bench=. -benchmem

# 結果：
# BenchmarkSliceAppend-8           50000    30000 ns/op    16384 B/op    10 allocs/op
# BenchmarkSlicePrealloc-8        100000    10000 ns/op     8192 B/op     1 allocs/op
# BenchmarkMapInsert-8             30000    40000 ns/op    32768 B/op     5 allocs/op
# BenchmarkMapPrealloc-8           50000    25000 ns/op    16384 B/op     1 allocs/op
# BenchmarkLinkedList-8            10000   100000 ns/op    65536 B/op  1000 allocs/op
# BenchmarkSliceSearch-8            5000   300000 ns/op        0 B/op     0 allocs/op
# BenchmarkMapSearch-8           1000000     1000 ns/op        0 B/op     0 allocs/op
```

## 6. 性能分析 (Profiling)

### CPU Profiling

```bash
# 生成 CPU profile
go test -bench=. -cpuprofile=cpu.prof

# 分析 profile
go tool pprof cpu.prof

# 在交互式模式中：
# (pprof) top10        # 顯示最耗時的 10 個函數
# (pprof) list funcName # 顯示函數的詳細信息
# (pprof) web          # 生成調用圖（需要 graphviz）

# 直接生成網頁報告
go tool pprof -http=:8080 cpu.prof
```

### Memory Profiling

```bash
# 生成內存 profile
go test -bench=. -memprofile=mem.prof

# 分析 profile
go tool pprof mem.prof

# 查看分配最多內存的函數
# (pprof) top10
# (pprof) list funcName

# 網頁報告
go tool pprof -http=:8080 mem.prof
```

### Block Profiling

```bash
# 生成阻塞 profile
go test -bench=. -blockprofile=block.prof

# 分析
go tool pprof block.prof
```

## 7. 實戰技巧

### 避免編譯器優化

```go
package benchmark

import "testing"

var result int

func BenchmarkWithoutOptimization(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = expensiveComputation()
    }
    // 將結果賦值給包級變量，防止編譯器優化掉整個循環
    result = r
}

func BenchmarkFibonacci(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = fibonacci(20)
    }
    result = r
}
```

### 基準測試最佳實踐

```go
package benchmark

import (
    "math/rand"
    "testing"
)

// 1. 使用 ResetTimer 排除 Setup 時間
func BenchmarkGoodExample(b *testing.B) {
    data := generateLargeDataset()
    b.ResetTimer()

    for i := 0; i < b.N; i++ {
        process(data)
    }
}

// 2. 使用 b.N 作為循環次數
func BenchmarkCorrectLoop(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // 測試代碼
    }
}

// 3. 避免在循環內做 Setup
func BenchmarkBadExample(b *testing.B) {
    for i := 0; i < b.N; i++ {
        data := generateLargeDataset() // 不好：每次都生成
        process(data)
    }
}

// 4. 對於隨機數據，使用固定種子
func BenchmarkWithRandomData(b *testing.B) {
    r := rand.New(rand.NewSource(42))

    for i := 0; i < b.N; i++ {
        data := r.Intn(1000)
        process(data)
    }
}

// 5. 使用子基準測試比較不同情況
func BenchmarkComparison(b *testing.B) {
    sizes := []int{10, 100, 1000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("Size%d", size), func(b *testing.B) {
            data := make([]int, size)
            b.ResetTimer()

            for i := 0; i < b.N; i++ {
                sort.Ints(data)
            }
        })
    }
}
```

## 重點總結

### Go vs benchmark.js

| 特性 | Go testing | benchmark.js |
|------|------------|--------------|
| **安裝** | 內置 | 需要安裝 |
| **運行** | go test -bench | node script |
| **並行測試** | RunParallel | N/A |
| **內存分析** | -benchmem | N/A |
| **統計分析** | benchstat | 內置 |
| **Profiling** | 集成 pprof | 需要其他工具 |

### 最佳實踐

1. **使用 -benchmem** 查看內存分配
2. **運行多次** 獲得穩定結果（-count=10）
3. **使用 ResetTimer** 排除 setup 時間
4. **使用子基準測試** 比較不同場景
5. **使用 profiling** 深入分析性能
6. **保存結果** 用於版本對比

### 性能優化流程

1. 編寫基準測試
2. 運行基準測試獲取基線
3. 使用 profiling 找到瓶頸
4. 優化代碼
5. 重新運行基準測試
6. 使用 benchstat 比較結果
7. 重複 3-6 直到滿意

## 練習題

1. **基礎**：比較不同排序算法的性能
2. **中級**：測試並優化一個緩存系統的性能
3. **高級**：對比不同併發模式的性能（channel vs mutex）
4. **實戰**：對一個 Web API 進行性能優化，使用基準測試驗證改進

## 下一步

下一章將學習表格驅動測試（Table-Driven Tests），這是 Go 中非常流行的測試模式。
