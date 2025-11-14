# 第十章：循環

## 章節概述

本章將全面介紹 Go 的循環結構。Go 只有一種循環關鍵字 `for`，但它可以實現多種循環模式。我們將對比 JavaScript 的 for、while、do-while、forEach 等多種循環方式。

## for 循環概覽

### JavaScript 的循環方式

```javascript
// JavaScript 有多種循環
// 1. for 循環
for (let i = 0; i < 5; i++) {
    console.log(i);
}

// 2. while 循環
let i = 0;
while (i < 5) {
    console.log(i);
    i++;
}

// 3. do-while 循環
let j = 0;
do {
    console.log(j);
    j++;
} while (j < 5);

// 4. for...of (數組)
for (let item of [1, 2, 3]) {
    console.log(item);
}

// 5. for...in (對象)
for (let key in {a: 1, b: 2}) {
    console.log(key);
}

// 6. forEach
[1, 2, 3].forEach(item => console.log(item));
```

### Go 的 for 循環

```go
package main

import "fmt"

func main() {
    // Go 只有 for，但它非常靈活！

    // 1. 經典 for 循環
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }

    // 2. while 風格
    i := 0
    for i < 5 {
        fmt.Println(i)
        i++
    }

    // 3. 無限循環
    // for {
    //     fmt.Println("Forever")
    //     break  // 需要手動跳出
    // }

    // 4. range 遍歷
    for index, value := range []int{1, 2, 3} {
        fmt.Println(index, value)
    }

    // Go 沒有 do-while
    // 但可以用 for + break 實現
}
```

## 經典 for 循環

### 基本語法

```go
package main

import "fmt"

func main() {
    // for 初始化; 條件; 後置語句 {
    //     循環體
    // }

    for i := 0; i < 5; i++ {
        fmt.Println(i)  // 0 1 2 3 4
    }

    // 多個變量
    for i, j := 0, 10; i < j; i, j = i+1, j-1 {
        fmt.Printf("i=%d, j=%d\n", i, j)
    }

    // 省略初始化（變量已存在）
    i := 0
    for ; i < 5; i++ {
        fmt.Println(i)
    }

    // 省略後置語句
    i = 0
    for i < 5 {
        fmt.Println(i)
        i++
    }

    // 所有都省略 = 無限循環
    // for {
    //     fmt.Println("Forever")
    // }
}
```

**對比 JavaScript:**
```javascript
// JavaScript for 循環
for (let i = 0; i < 5; i++) {
    console.log(i);
}

// 多個變量
for (let i = 0, j = 10; i < j; i++, j--) {
    console.log(`i=${i}, j=${j}`);
}

// 省略部分
let i = 0;
for (; i < 5; i++) {
    console.log(i);
}

// 無限循環
// for (;;) {
//     console.log("Forever");
// }

// 或 while(true)
```

### 注意事項

```go
package main

import "fmt"

func main() {
    // ❌ 條件不需要括號
    // for (i := 0; i < 5; i++) { }  // 語法錯誤

    // ✅ 大括號必須有
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }

    // ❌ 單行也不能省略大括號
    // for i := 0; i < 5; i++
    //     fmt.Println(i)  // 語法錯誤
}
```

## while 風格循環

### Go 的 while

```go
package main

import "fmt"

func main() {
    // Go 沒有 while 關鍵字
    // 使用 for + 條件

    i := 0
    for i < 5 {
        fmt.Println(i)
        i++
    }

    // 等價於 JavaScript 的 while
}
```

**對比 JavaScript:**
```javascript
let i = 0;
while (i < 5) {
    console.log(i);
    i++;
}
```

### 實際應用

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

func main() {
    reader := bufio.NewReader(os.Stdin)

    // 讀取輸入直到用戶輸入 "quit"
    for {
        fmt.Print("Enter command (quit to exit): ")
        input, _ := reader.ReadString('\n')
        input = strings.TrimSpace(input)

        if input == "quit" {
            break
        }

        fmt.Println("You entered:", input)
    }
}
```

**對比 JavaScript:**
```javascript
const readline = require('readline');
const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout
});

function prompt() {
    rl.question('Enter command (quit to exit): ', (input) => {
        input = input.trim();

        if (input === 'quit') {
            rl.close();
            return;
        }

        console.log('You entered:', input);
        prompt();  // 遞歸調用
    });
}

prompt();
```

## do-while 的替代方案

### Go 沒有 do-while

```go
package main

import "fmt"

func main() {
    // JavaScript 的 do-while
    // do {
    //     // 至少執行一次
    // } while (condition)

    // Go 的替代方案 1：for + break
    for {
        // 循環體（至少執行一次）
        fmt.Println("Execute once")

        if !condition() {
            break
        }
    }

    // 替代方案 2：先執行後判斷
    execute()
    for condition() {
        execute()
    }
}

func condition() bool {
    return false
}

func execute() {
    fmt.Println("Execute")
}
```

**對比 JavaScript:**
```javascript
// JavaScript do-while
let i = 0;
do {
    console.log(i);
    i++;
} while (i < 5);

// 至少執行一次
let x = 10;
do {
    console.log("Executes once");
} while (x < 5);  // 條件為 false，但已執行過一次
```

## range 遍歷

這是 Go 最常用的遍歷方式！

### 遍歷數組/切片

```go
package main

import "fmt"

func main() {
    numbers := []int{10, 20, 30, 40, 50}

    // 同時獲取索引和值
    for index, value := range numbers {
        fmt.Printf("Index: %d, Value: %d\n", index, value)
    }

    // 只要值（忽略索引）
    for _, value := range numbers {
        fmt.Println(value)
    }

    // 只要索引
    for index := range numbers {
        fmt.Println(index)
    }

    // 都不要（只是執行 N 次）
    for range numbers {
        fmt.Println("Loop")
    }
}
```

**對比 JavaScript:**
```javascript
const numbers = [10, 20, 30, 40, 50];

// for...of（只有值）
for (let value of numbers) {
    console.log(value);
}

// forEach（索引和值）
numbers.forEach((value, index) => {
    console.log(`Index: ${index}, Value: ${value}`);
});

// 經典 for（索引和值）
for (let i = 0; i < numbers.length; i++) {
    console.log(`Index: ${i}, Value: ${numbers[i]}`);
}

// entries()（索引和值）
for (let [index, value] of numbers.entries()) {
    console.log(`Index: ${index}, Value: ${value}`);
}
```

### 遍歷 Map

```go
package main

import "fmt"

func main() {
    ages := map[string]int{
        "Alice": 30,
        "Bob":   25,
        "Charlie": 35,
    }

    // 遍歷鍵值對
    for key, value := range ages {
        fmt.Printf("%s: %d\n", key, value)
    }

    // 只要鍵
    for key := range ages {
        fmt.Println(key)
    }

    // 注意：map 遍歷順序是隨機的！
}
```

**對比 JavaScript:**
```javascript
const ages = {
    "Alice": 30,
    "Bob": 25,
    "Charlie": 35
};

// for...in（對象鍵）
for (let key in ages) {
    console.log(`${key}: ${ages[key]}`);
}

// Object.entries()
for (let [key, value] of Object.entries(ages)) {
    console.log(`${key}: ${value}`);
}

// ES6 Map
const agesMap = new Map([
    ["Alice", 30],
    ["Bob", 25]
]);

for (let [key, value] of agesMap) {
    console.log(`${key}: ${value}`);
}
```

### 遍歷字符串

```go
package main

import "fmt"

func main() {
    str := "Hello, 世界"

    // range 遍歷 rune（字符）
    for index, char := range str {
        fmt.Printf("%d: %c\n", index, char)
    }
    // 輸出：
    // 0: H
    // 1: e
    // 2: l
    // 3: l
    // 4: o
    // 5: ,
    // 6:
    // 7: 世
    // 10: 界（注意索引跳躍！）

    // 遍歷字節
    for i := 0; i < len(str); i++ {
        fmt.Printf("%d: %c\n", i, str[i])
    }
    // 中文會顯示亂碼
}
```

**對比 JavaScript:**
```javascript
const str = "Hello, 世界";

// for...of（正確處理 Unicode）
for (let char of str) {
    console.log(char);
}

// 經典 for
for (let i = 0; i < str.length; i++) {
    console.log(str[i]);
}
```

### 遍歷通道（Channel）

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3
    close(ch)

    // range 遍歷 channel
    for value := range ch {
        fmt.Println(value)
    }
    // channel 關閉後自動退出
}
```

**JavaScript 沒有直接對應的概念（但有 async iterators）。**

## break 和 continue

### break

```go
package main

import "fmt"

func main() {
    // break 跳出當前循環
    for i := 0; i < 10; i++ {
        if i == 5 {
            break  // 跳出循環
        }
        fmt.Println(i)  // 0 1 2 3 4
    }

    // 在 range 中使用
    numbers := []int{1, 2, 3, 4, 5}
    for _, num := range numbers {
        if num == 3 {
            break
        }
        fmt.Println(num)  // 1 2
    }
}
```

**對比 JavaScript:**
```javascript
// JavaScript break
for (let i = 0; i < 10; i++) {
    if (i === 5) {
        break;
    }
    console.log(i);  // 0 1 2 3 4
}

// forEach 不能使用 break！
[1, 2, 3, 4, 5].forEach(num => {
    if (num === 3) {
        // break;  // ❌ 語法錯誤
        return;   // ✅ 但只是跳過當前迭代，不是跳出循環
    }
    console.log(num);  // 1 2 4 5
});

// 需要用 some() 或 every() 模擬 break
[1, 2, 3, 4, 5].some(num => {
    if (num === 3) return true;  // 跳出
    console.log(num);  // 1 2
    return false;
});
```

### continue

```go
package main

import "fmt"

func main() {
    // continue 跳過當前迭代
    for i := 0; i < 5; i++ {
        if i == 2 {
            continue  // 跳過 i=2
        }
        fmt.Println(i)  // 0 1 3 4
    }

    // 在 range 中使用
    for i, num := range []int{1, 2, 3, 4, 5} {
        if num%2 == 0 {
            continue  // 跳過偶數
        }
        fmt.Printf("%d: %d\n", i, num)  // 0:1, 2:3, 4:5
    }
}
```

**對比 JavaScript:**
```javascript
// JavaScript continue
for (let i = 0; i < 5; i++) {
    if (i === 2) {
        continue;
    }
    console.log(i);  // 0 1 3 4
}

// forEach 使用 return（類似 continue）
[1, 2, 3, 4, 5].forEach(num => {
    if (num % 2 === 0) {
        return;  // 跳過當前
    }
    console.log(num);  // 1 3 5
});
```

## 標籤和嵌套循環

### 標籤跳出

```go
package main

import "fmt"

func main() {
    // 使用標籤跳出外層循環
outer:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            fmt.Printf("i=%d, j=%d\n", i, j)
            if i == 1 && j == 1 {
                break outer  // 跳出外層循環
            }
        }
    }
    // 輸出：
    // i=0, j=0
    // i=0, j=1
    // i=0, j=2
    // i=1, j=0
    // i=1, j=1

    // continue 也可以使用標籤
outer2:
    for i := 0; i < 3; i++ {
        for j := 0; j < 3; j++ {
            if j == 1 {
                continue outer2  // 跳到外層循環的下一次迭代
            }
            fmt.Printf("i=%d, j=%d\n", i, j)
        }
    }
}
```

**對比 JavaScript:**
```javascript
// JavaScript 也支持標籤
outer:
for (let i = 0; i < 3; i++) {
    for (let j = 0; j < 3; j++) {
        console.log(`i=${i}, j=${j}`);
        if (i === 1 && j === 1) {
            break outer;  // 跳出外層
        }
    }
}

// continue 標籤
outer2:
for (let i = 0; i < 3; i++) {
    for (let j = 0; j < 3; j++) {
        if (j === 1) {
            continue outer2;
        }
        console.log(`i=${i}, j=${j}`);
    }
}
```

## 常見模式

### 1. 倒序遍歷

```go
package main

import "fmt"

func main() {
    numbers := []int{1, 2, 3, 4, 5}

    // 倒序
    for i := len(numbers) - 1; i >= 0; i-- {
        fmt.Println(numbers[i])  // 5 4 3 2 1
    }
}
```

**對比 JavaScript:**
```javascript
const numbers = [1, 2, 3, 4, 5];

for (let i = numbers.length - 1; i >= 0; i--) {
    console.log(numbers[i]);  // 5 4 3 2 1
}

// 或使用 reverse()
[...numbers].reverse().forEach(n => console.log(n));
```

### 2. 跳步遍歷

```go
package main

import "fmt"

func main() {
    // 每次增加 2
    for i := 0; i < 10; i += 2 {
        fmt.Println(i)  // 0 2 4 6 8
    }

    // 遍歷偶數索引
    numbers := []int{1, 2, 3, 4, 5, 6}
    for i := 0; i < len(numbers); i += 2 {
        fmt.Println(numbers[i])  // 1 3 5
    }
}
```

### 3. 無限循環 + break

```go
package main

import "fmt"

func main() {
    count := 0

    for {
        fmt.Println(count)
        count++

        if count >= 5 {
            break
        }
    }
}
```

**對比 JavaScript:**
```javascript
let count = 0;

while (true) {
    console.log(count);
    count++;

    if (count >= 5) {
        break;
    }
}
```

### 4. 範圍生成

```go
package main

import "fmt"

func main() {
    // 生成範圍
    for i := range make([]int, 5) {
        fmt.Println(i)  // 0 1 2 3 4
    }

    // 或更明確
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }
}
```

## 性能考慮

### range 複製

```go
package main

import "fmt"

type BigStruct struct {
    data [1000]int
}

func main() {
    items := []BigStruct{{}, {}, {}}

    // ❌ range 會複製值（性能問題）
    for _, item := range items {
        _ = item  // item 是副本
    }

    // ✅ 使用索引訪問（避免複製）
    for i := range items {
        _ = items[i]  // 直接訪問，不複製
    }

    // ✅ 或使用指針切片
    ptrs := []*BigStruct{&items[0], &items[1], &items[2]}
    for _, ptr := range ptrs {
        _ = ptr  // 只複製指針
    }
}
```

## 重點總結

### Go 循環特點

1. **只有 for**（但非常靈活）
2. **range** 是遍歷的首選方式
3. **沒有 while 和 do-while**（用 for 實現）
4. **條件不需要括號**
5. **大括號必須有**
6. **支持標籤跳轉**

### for vs while vs range

| 用途 | Go | JavaScript |
|------|-----|------------|
| **計數循環** | `for i := 0; i < 10; i++` | `for (let i = 0; i < 10; i++)` |
| **條件循環** | `for condition { }` | `while (condition) { }` |
| **無限循環** | `for { }` | `while (true) { }` |
| **遍歷數組** | `for _, v := range arr` | `for (let v of arr)` |
| **遍歷對象** | `for k, v := range map` | `for (let k in obj)` |
| **至少執行一次** | `for { if !cond { break } }` | `do { } while (cond)` |

### 最佳實踐

1. **優先使用 range**（代碼更清晰）
2. **注意 range 複製大結構體**
3. **使用有意義的變量名**（不只是 i, j）
4. **避免過深的嵌套循環**
5. **使用 break/continue 簡化邏輯**

## 練習題

### 基礎題

1. **循環基礎**
   - 用三種方式實現 1 到 10 的循環
   - 對比 for、while 風格、range
   - 測試 break 和 continue

2. **range 練習**
   - 遍歷切片、map、字符串
   - 只要索引、只要值、都要
   - 理解 _ 的用法

3. **嵌套循環**
   - 打印九九乘法表
   - 使用標籤跳出
   - 對比 Go 和 JavaScript

### 進階題

4. **性能對比**
   - 對比 range 和索引遍歷
   - 測試大結構體的複製開銷
   - 優化循環性能

5. **模式實現**
   - 實現 do-while 模式
   - 實現 forEach 功能
   - 實現 filter/map/reduce

6. **複雜遍歷**
   - 同時遍歷兩個切片
   - 實現 zip 功能
   - 處理不等長切片

### 實戰題

7. **數據處理**
   - 遍歷用戶列表進行過濾
   - 統計、求和、平均值
   - 對比函數式編程方式

8. **矩陣操作**
   - 遍歷二維數組
   - 實現矩陣轉置
   - 使用標籤處理嵌套

9. **對比實現**
   - 用 Go 和 JavaScript 實現相同的遍歷邏輯
   - 對比代碼簡潔性
   - 記錄 range 的優勢

## 下一章預告

下一章我們將學習 Go 的函數，包括：
- func 聲明 vs function/arrow function
- 多返回值
- 命名返回值
- 可變參數
- 閉包和匿名函數
- defer 語句

準備好學習 Go 的函數了嗎？
