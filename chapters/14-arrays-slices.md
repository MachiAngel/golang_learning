# Chapter 14: 數組與切片 (Arrays & Slices)

## 概述

在 Go 中，數組和切片是兩種不同的數據結構，這與 Node.js 中只有一種 Array 類型不同。數組是固定長度的，而切片是動態的、更靈活的數據結構。切片是 Go 中最常用的序列數據類型，類似於 JavaScript 的數組。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 動態數組 | `Array` | `slice` (切片) |
| 固定長度數組 | 不直接支持 | `array` (數組) |
| 長度可變 | `push`, `pop` 等方法 | `append` 函數 |
| 訪問元素 | `arr[i]` | `arr[i]` |
| 長度獲取 | `arr.length` | `len(arr)` |
| 切割操作 | `arr.slice(start, end)` | `arr[start:end]` |

## 詳細概念解釋

### 1. 數組 (Array)

數組是固定長度的序列，長度在編譯時確定，不能改變。

**特點：**
- 長度是類型的一部分：`[3]int` 和 `[4]int` 是不同類型
- 值類型：賦值或傳參會複製整個數組
- 性能好但不靈活

### 2. 切片 (Slice)

切片是對數組的引用，是動態的、靈活的數據結構。

**特點：**
- 動態長度：可以通過 `append` 增長
- 引用類型：傳遞切片不會複製底層數組
- 有三個屬性：指針、長度(len)、容量(cap)

### 3. 切片的底層結構

```
切片結構：
┌─────────┬────────┬──────────┐
│ pointer │  len   │   cap    │
└────┬────┴────────┴──────────┘
     │
     └──> 底層數組
```

- **指針**：指向底層數組的起始位置
- **長度 (len)**：切片中元素的數量
- **容量 (cap)**：從切片起始位置到底層數組末尾的元素數量

## 代碼示例

### 示例 1: 數組的創建和使用

**Go:**
```go
package main

import "fmt"

func main() {
    // 創建固定長度數組
    var arr1 [3]int
    fmt.Println("零值數組:", arr1) // [0 0 0]

    // 初始化數組
    arr2 := [3]int{1, 2, 3}
    fmt.Println("初始化數組:", arr2) // [1 2 3]

    // 讓編譯器計算長度
    arr3 := [...]int{1, 2, 3, 4, 5}
    fmt.Println("自動長度:", arr3) // [1 2 3 4 5]
    fmt.Println("長度:", len(arr3)) // 5

    // 指定索引初始化
    arr4 := [5]int{0: 10, 2: 20, 4: 30}
    fmt.Println("指定索引:", arr4) // [10 0 20 0 30]

    // 數組是值類型
    arr5 := arr2
    arr5[0] = 100
    fmt.Println("原數組:", arr2) // [1 2 3]
    fmt.Println("新數組:", arr5) // [100 2 3]
}
```

**Node.js 對比:**
```javascript
// JavaScript 沒有固定長度數組的概念
// 所有數組都是動態的

// 創建數組
const arr1 = [];
console.log("空數組:", arr1); // []

// 初始化數組
const arr2 = [1, 2, 3];
console.log("初始化數組:", arr2); // [1, 2, 3]

// JavaScript 數組可以動態增長
const arr3 = [1, 2, 3, 4, 5];
console.log("數組:", arr3);
console.log("長度:", arr3.length); // 5

// JavaScript 數組是引用類型
const arr4 = arr2;
arr4[0] = 100;
console.log("原數組:", arr2); // [100, 2, 3] - 被修改了！
console.log("新數組:", arr4); // [100, 2, 3]

// 如果要複製數組，需要使用展開運算符
const arr5 = [...arr2];
arr5[0] = 200;
console.log("原數組:", arr2); // [100, 2, 3]
console.log("複製數組:", arr5); // [200, 2, 3]
```

### 示例 2: 切片的創建和使用

**Go:**
```go
package main

import "fmt"

func main() {
    // 方法 1: 使用字面量創建
    slice1 := []int{1, 2, 3, 4, 5}
    fmt.Printf("slice1: %v, len=%d, cap=%d\n", slice1, len(slice1), cap(slice1))

    // 方法 2: 使用 make 創建
    slice2 := make([]int, 3)      // 長度和容量都是 3
    fmt.Printf("slice2: %v, len=%d, cap=%d\n", slice2, len(slice2), cap(slice2))

    slice3 := make([]int, 3, 5)   // 長度 3，容量 5
    fmt.Printf("slice3: %v, len=%d, cap=%d\n", slice3, len(slice3), cap(slice3))

    // 方法 3: 從數組創建切片
    arr := [5]int{1, 2, 3, 4, 5}
    slice4 := arr[1:4]  // 包含索引 1, 2, 3
    fmt.Printf("slice4: %v, len=%d, cap=%d\n", slice4, len(slice4), cap(slice4))

    // 切片是引用類型
    slice5 := slice1
    slice5[0] = 100
    fmt.Println("原切片:", slice1) // [100 2 3 4 5] - 被修改了
    fmt.Println("新切片:", slice5) // [100 2 3 4 5]
}
```

**Node.js 對比:**
```javascript
// 方法 1: 字面量創建
const arr1 = [1, 2, 3, 4, 5];
console.log("arr1:", arr1, "length:", arr1.length);

// 方法 2: 使用 Array 構造函數
const arr2 = new Array(3);  // 創建長度為 3 的數組
arr2.fill(0);               // 填充零值
console.log("arr2:", arr2, "length:", arr2.length);

// 方法 3: 使用 Array.from
const arr3 = Array.from({ length: 3 }, () => 0);
console.log("arr3:", arr3, "length:", arr3.length);

// 方法 4: 使用 slice 複製部分數組
const original = [1, 2, 3, 4, 5];
const arr4 = original.slice(1, 4);  // 包含索引 1, 2, 3
console.log("arr4:", arr4, "length:", arr4.length);

// JavaScript 數組是引用類型
const arr5 = arr1;
arr5[0] = 100;
console.log("原數組:", arr1); // [100, 2, 3, 4, 5]
console.log("新數組:", arr5); // [100, 2, 3, 4, 5]
```

### 示例 3: 切片操作 - 切割

**Go:**
```go
package main

import "fmt"

func main() {
    slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}

    // 切片語法: slice[start:end] (不包含 end)
    fmt.Println("slice[2:5]:", slice[2:5])   // [2 3 4]
    fmt.Println("slice[:5]:", slice[:5])     // [0 1 2 3 4]
    fmt.Println("slice[5:]:", slice[5:])     // [5 6 7 8 9]
    fmt.Println("slice[:]:", slice[:])       // [0 1 2 3 4 5 6 7 8 9]

    // 三索引切片: slice[start:end:cap]
    s1 := slice[2:5:7]  // 從索引 2 到 5，容量到 7
    fmt.Printf("s1: %v, len=%d, cap=%d\n", s1, len(s1), cap(s1))
    // s1: [2 3 4], len=3, cap=5

    // 切片共享底層數組
    s2 := slice[2:5]
    s2[0] = 100
    fmt.Println("修改後 slice:", slice)  // [0 1 100 3 4 5 6 7 8 9]
    fmt.Println("修改後 s2:", s2)        // [100 3 4]
}
```

**Node.js 對比:**
```javascript
const arr = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9];

// slice 方法: array.slice(start, end) (不包含 end)
console.log("arr.slice(2, 5):", arr.slice(2, 5));   // [2, 3, 4]
console.log("arr.slice(0, 5):", arr.slice(0, 5));   // [0, 1, 2, 3, 4]
console.log("arr.slice(5):", arr.slice(5));         // [5, 6, 7, 8, 9]
console.log("arr.slice():", arr.slice());           // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

// 負索引
console.log("arr.slice(-3):", arr.slice(-3));       // [7, 8, 9]
console.log("arr.slice(2, -2):", arr.slice(2, -2)); // [2, 3, 4, 5, 6, 7]

// JavaScript 的 slice 返回新數組，不共享數據
const arr2 = arr.slice(2, 5);
arr2[0] = 100;
console.log("修改後 arr:", arr);   // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9] - 未改變
console.log("修改後 arr2:", arr2); // [100, 3, 4]
```

### 示例 4: append 操作

**Go:**
```go
package main

import "fmt"

func main() {
    // append 添加元素
    var slice []int
    fmt.Printf("初始: %v, len=%d, cap=%d\n", slice, len(slice), cap(slice))

    slice = append(slice, 1)
    fmt.Printf("append 1: %v, len=%d, cap=%d\n", slice, len(slice), cap(slice))

    slice = append(slice, 2, 3, 4)
    fmt.Printf("append 2,3,4: %v, len=%d, cap=%d\n", slice, len(slice), cap(slice))

    // append 另一個切片
    slice2 := []int{5, 6, 7}
    slice = append(slice, slice2...)  // ... 展開切片
    fmt.Printf("append slice2: %v, len=%d, cap=%d\n", slice, len(slice), cap(slice))

    // 容量增長機制
    s := make([]int, 0)
    for i := 0; i < 10; i++ {
        s = append(s, i)
        fmt.Printf("len=%d, cap=%d\n", len(s), cap(s))
    }
    // 觀察容量如何增長：通常是翻倍策略
}
```

**Node.js 對比:**
```javascript
// push 添加元素
let arr = [];
console.log("初始:", arr, "length:", arr.length);

arr.push(1);
console.log("push 1:", arr, "length:", arr.length);

arr.push(2, 3, 4);
console.log("push 2,3,4:", arr, "length:", arr.length);

// 合併數組
const arr2 = [5, 6, 7];
arr.push(...arr2);  // 使用展開運算符
console.log("push arr2:", arr, "length:", arr.length);

// 或使用 concat
arr = arr.concat(arr2);
console.log("concat arr2:", arr, "length:", arr.length);

// JavaScript 數組會自動增長，沒有容量概念
const s = [];
for (let i = 0; i < 10; i++) {
    s.push(i);
    console.log("length:", s.length);
}
```

### 示例 5: 切片的複製

**Go:**
```go
package main

import "fmt"

func main() {
    source := []int{1, 2, 3, 4, 5}

    // 方法 1: 使用 copy 函數
    dest1 := make([]int, len(source))
    n := copy(dest1, source)
    fmt.Printf("複製了 %d 個元素\n", n)
    fmt.Println("source:", source)
    fmt.Println("dest1:", dest1)

    dest1[0] = 100
    fmt.Println("修改後 source:", source) // [1 2 3 4 5] - 未改變
    fmt.Println("修改後 dest1:", dest1)   // [100 2 3 4 5]

    // 方法 2: 部分複製
    dest2 := make([]int, 3)
    copy(dest2, source)
    fmt.Println("部分複製:", dest2) // [1 2 3]

    // 方法 3: 使用 append
    dest3 := append([]int{}, source...)
    fmt.Println("append 複製:", dest3) // [1 2 3 4 5]

    // copy 的目標長度不足時
    dest4 := make([]int, 2)
    copy(dest4, source)
    fmt.Println("長度不足:", dest4) // [1 2] - 只複製了前兩個
}
```

**Node.js 對比:**
```javascript
const source = [1, 2, 3, 4, 5];

// 方法 1: 使用展開運算符
const dest1 = [...source];
console.log("source:", source);
console.log("dest1:", dest1);

dest1[0] = 100;
console.log("修改後 source:", source); // [1, 2, 3, 4, 5]
console.log("修改後 dest1:", dest1);   // [100, 2, 3, 4, 5]

// 方法 2: 使用 slice
const dest2 = source.slice();
console.log("slice 複製:", dest2);

// 方法 3: 使用 Array.from
const dest3 = Array.from(source);
console.log("Array.from 複製:", dest3);

// 方法 4: 部分複製
const dest4 = source.slice(0, 3);
console.log("部分複製:", dest4); // [1, 2, 3]

// 方法 5: 使用 concat
const dest5 = [].concat(source);
console.log("concat 複製:", dest5);
```

### 示例 6: 切片常見操作

**Go:**
```go
package main

import "fmt"

func main() {
    slice := []int{1, 2, 3, 4, 5}

    // 1. 刪除元素
    // 刪除索引 2 的元素
    index := 2
    slice = append(slice[:index], slice[index+1:]...)
    fmt.Println("刪除索引 2:", slice) // [1 2 4 5]

    // 2. 插入元素
    // 在索引 2 插入 99
    slice = []int{1, 2, 3, 4, 5}
    index = 2
    value := 99
    slice = append(slice[:index], append([]int{value}, slice[index:]...)...)
    fmt.Println("插入 99:", slice) // [1 2 99 3 4 5]

    // 3. 過濾元素
    slice = []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    var evens []int
    for _, v := range slice {
        if v%2 == 0 {
            evens = append(evens, v)
        }
    }
    fmt.Println("偶數:", evens) // [2 4 6 8 10]

    // 4. 反轉切片
    slice = []int{1, 2, 3, 4, 5}
    for i, j := 0, len(slice)-1; i < j; i, j = i+1, j-1 {
        slice[i], slice[j] = slice[j], slice[i]
    }
    fmt.Println("反轉:", slice) // [5 4 3 2 1]
}
```

**Node.js 對比:**
```javascript
let arr = [1, 2, 3, 4, 5];

// 1. 刪除元素
const index = 2;
arr.splice(index, 1);
console.log("刪除索引 2:", arr); // [1, 2, 4, 5]

// 2. 插入元素
arr = [1, 2, 3, 4, 5];
arr.splice(2, 0, 99);
console.log("插入 99:", arr); // [1, 2, 99, 3, 4, 5]

// 3. 過濾元素
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
const evens = arr.filter(v => v % 2 === 0);
console.log("偶數:", evens); // [2, 4, 6, 8, 10]

// 4. 反轉數組
arr = [1, 2, 3, 4, 5];
arr.reverse();
console.log("反轉:", arr); // [5, 4, 3, 2, 1]

// 其他常用方法
arr = [1, 2, 3, 4, 5];
console.log("map:", arr.map(x => x * 2));        // [2, 4, 6, 8, 10]
console.log("reduce:", arr.reduce((a, b) => a + b)); // 15
console.log("find:", arr.find(x => x > 3));      // 4
console.log("some:", arr.some(x => x > 3));      // true
console.log("every:", arr.every(x => x > 0));    // true
```

### 示例 7: 多維切片

**Go:**
```go
package main

import "fmt"

func main() {
    // 創建二維切片
    matrix := [][]int{
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9},
    }

    fmt.Println("二維切片:")
    for i, row := range matrix {
        for j, val := range row {
            fmt.Printf("matrix[%d][%d] = %d ", i, j, val)
        }
        fmt.Println()
    }

    // 動態創建二維切片
    rows, cols := 3, 4
    matrix2 := make([][]int, rows)
    for i := range matrix2 {
        matrix2[i] = make([]int, cols)
    }

    // 填充數據
    counter := 1
    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            matrix2[i][j] = counter
            counter++
        }
    }

    fmt.Println("\n動態二維切片:")
    for _, row := range matrix2 {
        fmt.Println(row)
    }
}
```

**Node.js 對比:**
```javascript
// 創建二維數組
const matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
];

console.log("二維數組:");
for (let i = 0; i < matrix.length; i++) {
    for (let j = 0; j < matrix[i].length; j++) {
        process.stdout.write(`matrix[${i}][${j}] = ${matrix[i][j]} `);
    }
    console.log();
}

// 動態創建二維數組
const rows = 3, cols = 4;
const matrix2 = Array.from({ length: rows }, () =>
    Array.from({ length: cols }, () => 0)
);

// 填充數據
let counter = 1;
for (let i = 0; i < rows; i++) {
    for (let j = 0; j < cols; j++) {
        matrix2[i][j] = counter++;
    }
}

console.log("\n動態二維數組:");
for (const row of matrix2) {
    console.log(row);
}
```

## 重點總結

### 數組 vs 切片

| 特性 | 數組 | 切片 |
|------|------|------|
| 長度 | 固定，編譯時確定 | 動態，可增長 |
| 類型 | 值類型 | 引用類型 |
| 性能 | 略高 | 略低（有間接層） |
| 使用場景 | 固定數量的數據 | 絕大多數情況 |
| 聲明 | `[3]int` | `[]int` |

### 切片最佳實踐

1. **優先使用切片**：除非有特殊需求，否則使用切片而非數組
2. **預分配容量**：如果知道大小，使用 `make([]T, 0, capacity)` 減少擴容
3. **注意切片共享**：切片操作可能共享底層數組，修改時要小心
4. **使用 copy 複製**：需要獨立副本時使用 `copy` 函數
5. **避免內存洩漏**：大切片的小切片可能持有整個底層數組

### 關鍵差異

| Go | Node.js | 說明 |
|-----|---------|------|
| `len(slice)` | `array.length` | 獲取長度 |
| `cap(slice)` | 無對應概念 | 獲取容量 |
| `append(slice, item)` | `array.push(item)` | 添加元素 |
| `slice[start:end]` | `array.slice(start, end)` | 切割 |
| `copy(dest, src)` | `[...src]` | 複製 |
| `make([]T, len, cap)` | `new Array(len)` | 創建 |

## 練習題

### 練習 1: 切片基本操作
編寫一個程序，實現以下功能：
1. 創建一個整數切片，包含 1-10
2. 刪除索引為 5 的元素
3. 在索引 3 插入數字 99
4. 輸出結果

### 練習 2: 切片去重
編寫一個函數 `removeDuplicates`，接收一個整數切片，返回去重後的新切片。

**示例：**
```go
input := []int{1, 2, 2, 3, 4, 4, 5}
result := removeDuplicates(input)
// result: [1, 2, 3, 4, 5]
```

### 練習 3: 合併排序切片
編寫一個函數 `mergeSorted`，合併兩個已排序的切片，返回一個新的排序切片。

**示例：**
```go
slice1 := []int{1, 3, 5, 7}
slice2 := []int{2, 4, 6, 8}
result := mergeSorted(slice1, slice2)
// result: [1, 2, 3, 4, 5, 6, 7, 8]
```

### 練習 4: 矩陣轉置
編寫一個函數 `transpose`，實現矩陣轉置（行列互換）。

**示例：**
```go
matrix := [][]int{
    {1, 2, 3},
    {4, 5, 6},
}
result := transpose(matrix)
// result: [[1, 4], [2, 5], [3, 6]]
```

### 練習 5: 切片容量分析
編寫程序觀察切片的容量增長規律：
1. 創建一個空切片
2. 循環 append 20 個元素
3. 每次 append 後輸出長度和容量
4. 分析容量的增長模式

### 答案提示

**練習 1:**
```go
func main() {
    slice := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    // 刪除索引 5
    slice = append(slice[:5], slice[6:]...)
    // 在索引 3 插入 99
    slice = append(slice[:3], append([]int{99}, slice[3:]...)...)
    fmt.Println(slice)
}
```

**練習 2:**
```go
func removeDuplicates(slice []int) []int {
    seen := make(map[int]bool)
    result := []int{}
    for _, val := range slice {
        if !seen[val] {
            seen[val] = true
            result = append(result, val)
        }
    }
    return result
}
```

**練習 3:**
```go
func mergeSorted(s1, s2 []int) []int {
    result := make([]int, 0, len(s1)+len(s2))
    i, j := 0, 0
    for i < len(s1) && j < len(s2) {
        if s1[i] < s2[j] {
            result = append(result, s1[i])
            i++
        } else {
            result = append(result, s2[j])
            j++
        }
    }
    result = append(result, s1[i:]...)
    result = append(result, s2[j:]...)
    return result
}
```

**練習 4:**
```go
func transpose(matrix [][]int) [][]int {
    if len(matrix) == 0 {
        return [][]int{}
    }
    rows, cols := len(matrix), len(matrix[0])
    result := make([][]int, cols)
    for i := range result {
        result[i] = make([]int, rows)
        for j := range result[i] {
            result[i][j] = matrix[j][i]
        }
    }
    return result
}
```
