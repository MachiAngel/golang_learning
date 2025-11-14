# Chapter 15: Map 數據結構

## 概述

Map 是 Go 中的鍵值對數據結構，類似於 JavaScript 的 Object 或 Map。Map 提供快速的查找、插入和刪除操作，是處理關聯數據的理想選擇。與 JavaScript 不同，Go 的 map 鍵可以是任何可比較的類型。

## Node.js 與 Go 的對比

| 特性 | Node.js | Go |
|------|---------|-----|
| 基本對象 | `{}` 或 `Object` | `map[KeyType]ValueType` |
| Map 類型 | `Map` | `map` |
| 創建 | `{}`, `new Map()` | `make(map[K]V)` |
| 訪問元素 | `obj[key]`, `map.get(key)` | `m[key]` |
| 設置元素 | `obj[key] = value`, `map.set(k, v)` | `m[key] = value` |
| 刪除元素 | `delete obj[key]`, `map.delete(key)` | `delete(m, key)` |
| 檢查存在 | `key in obj`, `map.has(key)` | `value, ok := m[key]` |
| 獲取長度 | `Object.keys(obj).length`, `map.size` | `len(m)` |

## 詳細概念解釋

### 1. Map 的特性

- **引用類型**：map 是引用類型，傳遞 map 不會複製數據
- **無序**：map 的迭代順序是隨機的
- **nil map**：未初始化的 map 是 nil，不能直接使用
- **併發不安全**：多個 goroutine 同時讀寫需要加鎖

### 2. Map 的鍵類型要求

鍵必須是可比較的類型：
- ✅ 可以：整數、浮點數、字符串、指針、結構體（所有字段可比較）、數組
- ❌ 不可以：切片、map、函數

### 3. Map 的零值

未初始化的 map 是 `nil`：
```go
var m map[string]int  // m == nil
// m["key"] = 1       // panic: assignment to entry in nil map
```

必須使用 `make` 或字面量初始化：
```go
m := make(map[string]int)  // 已初始化，可以使用
m["key"] = 1               // OK
```

## 代碼示例

### 示例 1: 創建和初始化 Map

**Go:**
```go
package main

import "fmt"

func main() {
    // 方法 1: 使用 make 創建
    map1 := make(map[string]int)
    fmt.Println("空 map:", map1) // map[]

    // 方法 2: 使用字面量創建
    map2 := map[string]int{
        "apple":  1,
        "banana": 2,
        "cherry": 3,
    }
    fmt.Println("初始化 map:", map2)

    // 方法 3: 聲明但不初始化（nil map）
    var map3 map[string]int
    fmt.Println("nil map:", map3)        // map[]
    fmt.Println("是否為 nil:", map3 == nil) // true
    // map3["key"] = 1  // panic!

    // 方法 4: 創建指定容量的 map
    map4 := make(map[string]int, 10)  // 預分配空間
    fmt.Println("指定容量 map:", map4)

    // 不同鍵類型的 map
    intMap := map[int]string{1: "one", 2: "two"}
    structMap := map[struct{ x, y int }]string{
        {1, 2}: "point A",
        {3, 4}: "point B",
    }

    fmt.Println("int key:", intMap)
    fmt.Println("struct key:", structMap)
}
```

**Node.js 對比:**
```javascript
// 方法 1: 使用對象字面量
const obj1 = {};
console.log("空對象:", obj1); // {}

// 方法 2: 使用對象初始化
const obj2 = {
    apple: 1,
    banana: 2,
    cherry: 3
};
console.log("初始化對象:", obj2);

// 方法 3: 使用 Map
const map1 = new Map();
console.log("空 Map:", map1); // Map(0) {}

const map2 = new Map([
    ['apple', 1],
    ['banana', 2],
    ['cherry', 3]
]);
console.log("初始化 Map:", map2);

// Map 可以使用任何類型作為鍵
const map3 = new Map();
map3.set(1, 'one');
map3.set(2, 'two');
console.log("數字鍵 Map:", map3);

// 使用對象作為鍵
const key1 = { x: 1, y: 2 };
const key2 = { x: 3, y: 4 };
const objMap = new Map([
    [key1, 'point A'],
    [key2, 'point B']
]);
console.log("對象鍵 Map:", objMap);
```

### 示例 2: Map 的基本操作

**Go:**
```go
package main

import "fmt"

func main() {
    // 創建 map
    scores := make(map[string]int)

    // 1. 設置值
    scores["Alice"] = 95
    scores["Bob"] = 87
    scores["Charlie"] = 92
    fmt.Println("設置後:", scores)

    // 2. 獲取值
    aliceScore := scores["Alice"]
    fmt.Println("Alice 的分數:", aliceScore)

    // 3. 獲取不存在的鍵（返回零值）
    davidScore := scores["David"]
    fmt.Println("David 的分數:", davidScore) // 0

    // 4. 檢查鍵是否存在（兩個返回值）
    score, exists := scores["Alice"]
    if exists {
        fmt.Printf("Alice 存在，分數: %d\n", score)
    }

    score, exists = scores["David"]
    if !exists {
        fmt.Println("David 不存在")
    }

    // 5. 更新值
    scores["Alice"] = 98
    fmt.Println("更新後 Alice:", scores["Alice"])

    // 6. 刪除鍵
    delete(scores, "Bob")
    fmt.Println("刪除 Bob 後:", scores)

    // 7. 獲取長度
    fmt.Println("map 長度:", len(scores))

    // 8. 遍歷 map
    fmt.Println("\n遍歷 map:")
    for name, score := range scores {
        fmt.Printf("%s: %d\n", name, score)
    }
}
```

**Node.js 對比:**
```javascript
// 使用 Object
const scores = {};

// 1. 設置值
scores.Alice = 95;
scores['Bob'] = 87;
scores.Charlie = 92;
console.log("設置後:", scores);

// 2. 獲取值
const aliceScore = scores.Alice;
console.log("Alice 的分數:", aliceScore);

// 3. 獲取不存在的鍵（返回 undefined）
const davidScore = scores.David;
console.log("David 的分數:", davidScore); // undefined

// 4. 檢查鍵是否存在
if ('Alice' in scores) {
    console.log("Alice 存在，分數:", scores.Alice);
}

if (!('David' in scores)) {
    console.log("David 不存在");
}

// 5. 更新值
scores.Alice = 98;
console.log("更新後 Alice:", scores.Alice);

// 6. 刪除鍵
delete scores.Bob;
console.log("刪除 Bob 後:", scores);

// 7. 獲取長度
console.log("對象大小:", Object.keys(scores).length);

// 8. 遍歷對象
console.log("\n遍歷對象:");
for (const [name, score] of Object.entries(scores)) {
    console.log(`${name}: ${score}`);
}

console.log("\n========== 使用 Map ==========\n");

// 使用 Map
const scoresMap = new Map();

// 1. 設置值
scoresMap.set('Alice', 95);
scoresMap.set('Bob', 87);
scoresMap.set('Charlie', 92);
console.log("設置後:", scoresMap);

// 2. 獲取值
console.log("Alice 的分數:", scoresMap.get('Alice'));

// 3. 獲取不存在的鍵
console.log("David 的分數:", scoresMap.get('David')); // undefined

// 4. 檢查鍵是否存在
if (scoresMap.has('Alice')) {
    console.log("Alice 存在，分數:", scoresMap.get('Alice'));
}

// 5. 更新值
scoresMap.set('Alice', 98);
console.log("更新後 Alice:", scoresMap.get('Alice'));

// 6. 刪除鍵
scoresMap.delete('Bob');
console.log("刪除 Bob 後:", scoresMap);

// 7. 獲取長度
console.log("Map 大小:", scoresMap.size);

// 8. 遍歷 Map
console.log("\n遍歷 Map:");
for (const [name, score] of scoresMap) {
    console.log(`${name}: ${score}`);
}
```

### 示例 3: Map 作為集合（Set）

**Go:**
```go
package main

import "fmt"

func main() {
    // Go 沒有內建的 Set，可以用 map[T]bool 實現
    set := make(map[string]bool)

    // 添加元素
    set["apple"] = true
    set["banana"] = true
    set["cherry"] = true

    // 檢查元素是否存在
    if set["apple"] {
        fmt.Println("apple 存在於集合中")
    }

    if !set["durian"] {
        fmt.Println("durian 不存在於集合中")
    }

    // 刪除元素
    delete(set, "banana")

    // 遍歷集合
    fmt.Println("\n集合元素:")
    for item := range set {
        fmt.Println(item)
    }

    // 集合操作：並集
    set1 := map[string]bool{"a": true, "b": true, "c": true}
    set2 := map[string]bool{"b": true, "c": true, "d": true}

    union := make(map[string]bool)
    for k := range set1 {
        union[k] = true
    }
    for k := range set2 {
        union[k] = true
    }
    fmt.Println("\n並集:", union)

    // 集合操作：交集
    intersection := make(map[string]bool)
    for k := range set1 {
        if set2[k] {
            intersection[k] = true
        }
    }
    fmt.Println("交集:", intersection)

    // 集合操作：差集
    difference := make(map[string]bool)
    for k := range set1 {
        if !set2[k] {
            difference[k] = true
        }
    }
    fmt.Println("差集 (set1 - set2):", difference)
}
```

**Node.js 對比:**
```javascript
// JavaScript 有內建的 Set
const set = new Set();

// 添加元素
set.add('apple');
set.add('banana');
set.add('cherry');

// 檢查元素是否存在
if (set.has('apple')) {
    console.log("apple 存在於集合中");
}

if (!set.has('durian')) {
    console.log("durian 不存在於集合中");
}

// 刪除元素
set.delete('banana');

// 遍歷集合
console.log("\n集合元素:");
for (const item of set) {
    console.log(item);
}

// 集合操作
const set1 = new Set(['a', 'b', 'c']);
const set2 = new Set(['b', 'c', 'd']);

// 並集
const union = new Set([...set1, ...set2]);
console.log("\n並集:", union);

// 交集
const intersection = new Set([...set1].filter(x => set2.has(x)));
console.log("交集:", intersection);

// 差集
const difference = new Set([...set1].filter(x => !set2.has(x)));
console.log("差集 (set1 - set2):", difference);

// Set 的其他方法
console.log("\nSet 大小:", set.size);
set.clear();
console.log("清空後:", set);
```

### 示例 4: Map 的引用特性

**Go:**
```go
package main

import "fmt"

func main() {
    // Map 是引用類型
    original := map[string]int{
        "a": 1,
        "b": 2,
        "c": 3,
    }

    // 賦值不會複製
    copy := original
    copy["a"] = 100

    fmt.Println("原始 map:", original) // map[a:100 b:2 c:3]
    fmt.Println("複製 map:", copy)     // map[a:100 b:2 c:3]

    // 函數參數傳遞也是引用
    modifyMap(original)
    fmt.Println("函數修改後:", original) // map[a:100 b:200 c:3 d:4]

    // 如果需要真正的複製，需要手動複製
    realCopy := make(map[string]int)
    for k, v := range original {
        realCopy[k] = v
    }
    realCopy["a"] = 999

    fmt.Println("原始 map:", original)  // 未改變
    fmt.Println("真正複製:", realCopy)  // a 被修改為 999
}

func modifyMap(m map[string]int) {
    m["b"] = 200
    m["d"] = 4
}
```

**Node.js 對比:**
```javascript
// Object 是引用類型
const original = {
    a: 1,
    b: 2,
    c: 3
};

// 賦值不會複製
const copy = original;
copy.a = 100;

console.log("原始對象:", original); // { a: 100, b: 2, c: 3 }
console.log("複製對象:", copy);     // { a: 100, b: 2, c: 3 }

// 函數參數傳遞也是引用
function modifyObject(obj) {
    obj.b = 200;
    obj.d = 4;
}

modifyObject(original);
console.log("函數修改後:", original); // { a: 100, b: 200, c: 3, d: 4 }

// 淺複製
const shallowCopy = { ...original };
shallowCopy.a = 999;
console.log("原始對象:", original);     // 未改變
console.log("淺複製對象:", shallowCopy); // a 被修改為 999

// 深複製（針對簡單對象）
const deepCopy = JSON.parse(JSON.stringify(original));
deepCopy.b = 888;
console.log("原始對象:", original);  // 未改變
console.log("深複製對象:", deepCopy); // b 被修改為 888

// Map 也是引用類型
const map1 = new Map([['a', 1], ['b', 2]]);
const map2 = map1;
map2.set('a', 100);
console.log("原始 Map:", map1.get('a')); // 100
console.log("複製 Map:", map2.get('a')); // 100

// Map 的真實複製
const realCopy = new Map(map1);
realCopy.set('a', 999);
console.log("原始 Map:", map1.get('a'));    // 100
console.log("真實複製 Map:", realCopy.get('a')); // 999
```

### 示例 5: 嵌套 Map

**Go:**
```go
package main

import "fmt"

func main() {
    // 嵌套 map：模擬二維結構
    // 例如：學生的各科成績
    grades := make(map[string]map[string]int)

    // 必須初始化內層 map
    grades["Alice"] = make(map[string]int)
    grades["Alice"]["Math"] = 95
    grades["Alice"]["English"] = 88

    grades["Bob"] = make(map[string]int)
    grades["Bob"]["Math"] = 87
    grades["Bob"]["English"] = 92

    // 訪問嵌套值
    fmt.Println("Alice 的數學成績:", grades["Alice"]["Math"])

    // 遍歷嵌套 map
    fmt.Println("\n所有成績:")
    for student, subjects := range grades {
        fmt.Printf("%s:\n", student)
        for subject, score := range subjects {
            fmt.Printf("  %s: %d\n", subject, score)
        }
    }

    // 檢查嵌套鍵是否存在
    if subjects, ok := grades["Charlie"]; ok {
        if score, ok := subjects["Math"]; ok {
            fmt.Println("Charlie 的數學成績:", score)
        }
    } else {
        fmt.Println("Charlie 不存在")
    }

    // 輔助函數：安全設置嵌套值
    setGrade := func(student, subject string, score int) {
        if grades[student] == nil {
            grades[student] = make(map[string]int)
        }
        grades[student][subject] = score
    }

    setGrade("Charlie", "Math", 90)
    setGrade("Charlie", "English", 85)

    fmt.Println("\n添加 Charlie 後:")
    for student, subjects := range grades {
        fmt.Printf("%s: %v\n", student, subjects)
    }
}
```

**Node.js 對比:**
```javascript
// 嵌套對象
const grades = {};

// JavaScript 對象可以直接設置嵌套屬性
grades.Alice = {};
grades.Alice.Math = 95;
grades.Alice.English = 88;

grades.Bob = {};
grades.Bob.Math = 87;
grades.Bob.English = 92;

// 訪問嵌套值
console.log("Alice 的數學成績:", grades.Alice.Math);

// 遍歷嵌套對象
console.log("\n所有成績:");
for (const [student, subjects] of Object.entries(grades)) {
    console.log(`${student}:`);
    for (const [subject, score] of Object.entries(subjects)) {
        console.log(`  ${subject}: ${score}`);
    }
}

// 檢查嵌套鍵是否存在
if (grades.Charlie && grades.Charlie.Math) {
    console.log("Charlie 的數學成績:", grades.Charlie.Math);
} else {
    console.log("Charlie 不存在");
}

// 使用可選鏈操作符（Optional Chaining）
const charlieScore = grades.Charlie?.Math;
console.log("Charlie 的數學成績:", charlieScore); // undefined

// 輔助函數：安全設置嵌套值
function setGrade(student, subject, score) {
    if (!grades[student]) {
        grades[student] = {};
    }
    grades[student][subject] = score;
}

setGrade('Charlie', 'Math', 90);
setGrade('Charlie', 'English', 85);

console.log("\n添加 Charlie 後:");
for (const [student, subjects] of Object.entries(grades)) {
    console.log(`${student}:`, subjects);
}

// 使用 Map 的嵌套版本
const gradesMap = new Map();

gradesMap.set('Alice', new Map([
    ['Math', 95],
    ['English', 88]
]));

gradesMap.set('Bob', new Map([
    ['Math', 87],
    ['English', 92]
]));

console.log("\n使用 Map:");
for (const [student, subjects] of gradesMap) {
    console.log(`${student}:`);
    for (const [subject, score] of subjects) {
        console.log(`  ${subject}: ${score}`);
    }
}
```

### 示例 6: Map 常見模式

**Go:**
```go
package main

import (
    "fmt"
    "sort"
)

func main() {
    // 1. 計數器模式
    text := "hello world"
    charCount := make(map[rune]int)

    for _, char := range text {
        charCount[char]++
    }

    fmt.Println("字符計數:", charCount)

    // 2. 分組模式
    words := []string{"apple", "banana", "apricot", "blueberry", "cherry"}
    grouped := make(map[rune][]string)

    for _, word := range words {
        firstChar := rune(word[0])
        grouped[firstChar] = append(grouped[firstChar], word)
    }

    fmt.Println("\n按首字母分組:", grouped)

    // 3. 緩存/記憶化模式
    cache := make(map[int]int)

    var fibonacci func(int) int
    fibonacci = func(n int) int {
        if n <= 1 {
            return n
        }
        if val, ok := cache[n]; ok {
            return val
        }
        result := fibonacci(n-1) + fibonacci(n-2)
        cache[n] = result
        return result
    }

    fmt.Println("\nFibonacci(10):", fibonacci(10))
    fmt.Println("緩存:", cache)

    // 4. 默認值模式
    config := map[string]string{
        "host": "localhost",
        "port": "8080",
    }

    getConfig := func(key, defaultValue string) string {
        if val, ok := config[key]; ok {
            return val
        }
        return defaultValue
    }

    fmt.Println("\nhost:", getConfig("host", "0.0.0.0"))
    fmt.Println("timeout:", getConfig("timeout", "30s"))

    // 5. 有序遍歷 map（需要先排序 keys）
    scores := map[string]int{
        "Charlie": 92,
        "Alice":   95,
        "Bob":     87,
    }

    // 提取並排序鍵
    keys := make([]string, 0, len(scores))
    for k := range scores {
        keys = append(keys, k)
    }
    sort.Strings(keys)

    fmt.Println("\n有序遍歷:")
    for _, k := range keys {
        fmt.Printf("%s: %d\n", k, scores[k])
    }
}
```

**Node.js 對比:**
```javascript
// 1. 計數器模式
const text = "hello world";
const charCount = {};

for (const char of text) {
    charCount[char] = (charCount[char] || 0) + 1;
}

console.log("字符計數:", charCount);

// 使用 Map
const charCountMap = new Map();
for (const char of text) {
    charCountMap.set(char, (charCountMap.get(char) || 0) + 1);
}
console.log("字符計數 (Map):", charCountMap);

// 2. 分組模式
const words = ["apple", "banana", "apricot", "blueberry", "cherry"];
const grouped = {};

for (const word of words) {
    const firstChar = word[0];
    if (!grouped[firstChar]) {
        grouped[firstChar] = [];
    }
    grouped[firstChar].push(word);
}

console.log("\n按首字母分組:", grouped);

// 使用 reduce
const grouped2 = words.reduce((acc, word) => {
    const firstChar = word[0];
    acc[firstChar] = acc[firstChar] || [];
    acc[firstChar].push(word);
    return acc;
}, {});

// 3. 緩存/記憶化模式
const cache = new Map();

function fibonacci(n) {
    if (n <= 1) return n;
    if (cache.has(n)) return cache.get(n);

    const result = fibonacci(n - 1) + fibonacci(n - 2);
    cache.set(n, result);
    return result;
}

console.log("\nFibonacci(10):", fibonacci(10));
console.log("緩存:", cache);

// 4. 默認值模式
const config = {
    host: 'localhost',
    port: '8080'
};

function getConfig(key, defaultValue) {
    return config[key] !== undefined ? config[key] : defaultValue;
}

// 或使用空值合併操作符
const getConfig2 = (key, defaultValue) => config[key] ?? defaultValue;

console.log("\nhost:", getConfig("host", "0.0.0.0"));
console.log("timeout:", getConfig("timeout", "30s"));

// 5. 有序遍歷對象
const scores = {
    Charlie: 92,
    Alice: 95,
    Bob: 87
};

// 排序鍵並遍歷
const sortedKeys = Object.keys(scores).sort();

console.log("\n有序遍歷:");
for (const key of sortedKeys) {
    console.log(`${key}: ${scores[key]}`);
}

// Map 保持插入順序
const scoresMap = new Map([
    ['Charlie', 92],
    ['Alice', 95],
    ['Bob', 87]
]);

console.log("\nMap 插入順序:");
for (const [name, score] of scoresMap) {
    console.log(`${name}: ${score}`);
}
```

## 重點總結

### Map 的關鍵特性

1. **引用類型**：賦值和傳參不會複製數據
2. **無序**：遍歷順序是隨機的（不同於 JavaScript 的 Map）
3. **nil map**：未初始化的 map 不能使用
4. **零值安全**：訪問不存在的鍵返回值類型的零值

### Map vs Object/Map (JavaScript)

| 特性 | Go map | JS Object | JS Map |
|------|--------|-----------|--------|
| 鍵類型 | 任何可比較類型 | 字符串/Symbol | 任何類型 |
| 有序性 | 無序 | 部分有序 | 保持插入順序 |
| 長度 | `len(m)` | `Object.keys(o).length` | `m.size` |
| 檢查存在 | `_, ok := m[k]` | `k in o` | `m.has(k)` |
| 刪除 | `delete(m, k)` | `delete o[k]` | `m.delete(k)` |

### 最佳實踐

1. **使用 make 初始化**：避免 nil map 錯誤
2. **檢查存在性**：使用 `value, ok := m[key]` 模式
3. **預分配容量**：大 map 使用 `make(map[K]V, capacity)` 提高性能
4. **不要依賴順序**：遍歷順序是隨機的
5. **併發訪問**：需要使用 `sync.Map` 或加鎖保護

### 常用模式

```go
// 初始化
m := make(map[K]V)

// 檢查存在
if v, ok := m[k]; ok {
    // k 存在，使用 v
}

// 默認值
v := m[k]  // 不存在返回零值

// 刪除
delete(m, k)

// 遍歷
for k, v := range m {
    // ...
}

// 計數
m[k]++

// 分組
m[k] = append(m[k], v)
```

## 練習題

### 練習 1: 單詞頻率統計
編寫一個函數 `wordFrequency`，統計字符串中每個單詞出現的次數。

**示例：**
```go
text := "hello world hello go world"
result := wordFrequency(text)
// result: map[hello:2 world:2 go:1]
```

### 練習 2: 兩個切片的交集
編寫一個函數 `intersection`，找出兩個整數切片的交集（不重複）。

**示例：**
```go
slice1 := []int{1, 2, 3, 4, 5}
slice2 := []int{3, 4, 5, 6, 7}
result := intersection(slice1, slice2)
// result: []int{3, 4, 5}
```

### 練習 3: 字符串分組
編寫一個函數 `groupAnagrams`，將字謎詞（相同字母不同順序）分組。

**示例：**
```go
words := []string{"eat", "tea", "tan", "ate", "nat", "bat"}
result := groupAnagrams(words)
// result: [[eat tea ate] [tan nat] [bat]]
```

### 練習 4: 實現 LRU 緩存
使用 map 實現一個簡單的 LRU（最近最少使用）緩存，支持 Get 和 Put 操作。

**提示：** 需要結合 map 和雙向鏈表。

### 練習 5: Map 深度合併
編寫一個函數 `mergeMaps`，深度合併兩個嵌套 map。

**示例：**
```go
map1 := map[string]interface{}{
    "a": 1,
    "b": map[string]interface{}{"x": 1, "y": 2},
}
map2 := map[string]interface{}{
    "b": map[string]interface{}{"y": 3, "z": 4},
    "c": 3,
}
result := mergeMaps(map1, map2)
// result: map[a:1 b:map[x:1 y:3 z:4] c:3]
```

### 答案提示

**練習 1:**
```go
func wordFrequency(text string) map[string]int {
    words := strings.Fields(text)
    freq := make(map[string]int)
    for _, word := range words {
        freq[word]++
    }
    return freq
}
```

**練習 2:**
```go
func intersection(slice1, slice2 []int) []int {
    set := make(map[int]bool)
    for _, v := range slice1 {
        set[v] = true
    }

    result := []int{}
    seen := make(map[int]bool)
    for _, v := range slice2 {
        if set[v] && !seen[v] {
            result = append(result, v)
            seen[v] = true
        }
    }
    return result
}
```

**練習 3:**
```go
func groupAnagrams(words []string) [][]string {
    groups := make(map[string][]string)

    for _, word := range words {
        // 將單詞排序作為鍵
        chars := []rune(word)
        sort.Slice(chars, func(i, j int) bool {
            return chars[i] < chars[j]
        })
        key := string(chars)
        groups[key] = append(groups[key], word)
    }

    result := [][]string{}
    for _, group := range groups {
        result = append(result, group)
    }
    return result
}
```
