# Chapter 29: time 包 - 時間處理

## 概述

`time` 包是 Go 標準庫中處理時間和日期的核心包。對於 Node.js 開發者來說，它相當於原生的 `Date` 對象以及流行的第三方庫（如 moment.js、dayjs、date-fns）的組合，但更加強大和類型安全。

## Node.js 與 Go 的對比

| 功能 | Node.js | Go |
|------|---------|-----|
| 當前時間 | `new Date()` | `time.Now()` |
| Unix 時間戳 | `Date.now()` | `time.Now().Unix()` |
| 創建時間 | `new Date(2024, 0, 1)` | `time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)` |
| 格式化 | `date.toISOString()` | `time.Format("2006-01-02")` |
| 解析 | `Date.parse(str)` | `time.Parse(layout, str)` |
| 時間差 | `date2 - date1` | `time2.Sub(time1)` |
| 添加時間 | `new Date(date.getTime() + ms)` | `time.Add(duration)` |
| 睡眠 | `await sleep(1000)` | `time.Sleep(time.Second)` |
| 定時器 | `setTimeout(fn, 1000)` | `time.After(time.Second)` |
| 比較 | `date1 < date2` | `time1.Before(time2)` |

## 詳細概念解釋

### 1. Time 類型

`time.Time` 是 Go 中表示時間點的類型，包含以下信息：
- 納秒級精度的時間戳
- 時區信息
- 單調時鐘（用於精確測量時間間隔）

### 2. Duration 類型

`time.Duration` 表示時間間隔，是 `int64` 類型的別名，單位為納秒。

常用常量：
```go
time.Nanosecond  // 1 納秒
time.Microsecond // 1000 納秒
time.Millisecond // 1000000 納秒
time.Second      // 1 秒
time.Minute      // 60 秒
time.Hour        // 60 分鐘
```

### 3. 時間格式化布局

Go 使用獨特的格式化方式：使用參考時間 `Mon Jan 2 15:04:05 MST 2006` 作為布局模板。

| 組件 | 參考值 | 說明 |
|------|--------|------|
| 年 | `2006` | 4 位年份 |
| 月 | `01` 或 `1` | 2 位或 1 位月份 |
| 月名 | `Jan` | 月份簡稱 |
| 月名 | `January` | 月份全稱 |
| 日 | `02` 或 `2` | 2 位或 1 位日期 |
| 星期 | `Mon` | 星期簡稱 |
| 星期 | `Monday` | 星期全稱 |
| 小時 | `15` | 24 小時制 |
| 小時 | `03` 或 `3` | 12 小時制 |
| 分鐘 | `04` | 分鐘 |
| 秒 | `05` | 秒 |
| AM/PM | `PM` | 上午/下午 |
| 時區 | `MST` | 時區簡稱 |
| 時區偏移 | `-0700` | 時區偏移 |

## 實際代碼示例

### 示例 1: 獲取當前時間

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 獲取當前時間
    now := time.Now()
    fmt.Println("Current time:", now)

    // 獲取各個組件
    fmt.Printf("Year: %d\n", now.Year())
    fmt.Printf("Month: %d (%s)\n", now.Month(), now.Month())
    fmt.Printf("Day: %d\n", now.Day())
    fmt.Printf("Hour: %d\n", now.Hour())
    fmt.Printf("Minute: %d\n", now.Minute())
    fmt.Printf("Second: %d\n", now.Second())
    fmt.Printf("Weekday: %s\n", now.Weekday())

    // Unix 時間戳
    fmt.Printf("Unix timestamp (seconds): %d\n", now.Unix())
    fmt.Printf("Unix timestamp (milliseconds): %d\n", now.UnixMilli())
    fmt.Printf("Unix timestamp (nanoseconds): %d\n", now.UnixNano())
}
```

**Node.js:**
```javascript
// 獲取當前時間
const now = new Date();
console.log("Current time:", now);

// 獲取各個組件
console.log(`Year: ${now.getFullYear()}`);
console.log(`Month: ${now.getMonth() + 1} (${now.toLocaleString('en', {month: 'long'})})`);
console.log(`Day: ${now.getDate()}`);
console.log(`Hour: ${now.getHours()}`);
console.log(`Minute: ${now.getMinutes()}`);
console.log(`Second: ${now.getSeconds()}`);
console.log(`Weekday: ${now.toLocaleString('en', {weekday: 'long'})}`);

// Unix 時間戳
console.log(`Unix timestamp (seconds): ${Math.floor(now.getTime() / 1000)}`);
console.log(`Unix timestamp (milliseconds): ${now.getTime()}`);
console.log(`Unix timestamp (nanoseconds): ${now.getTime() * 1000000}`);
```

### 示例 2: 創建特定時間

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 創建特定時間
    // time.Date(year, month, day, hour, min, sec, nsec, location)
    t1 := time.Date(2024, time.January, 1, 0, 0, 0, 0, time.UTC)
    fmt.Println("New Year 2024:", t1)

    // 使用本地時區
    t2 := time.Date(2024, 12, 25, 12, 30, 0, 0, time.Local)
    fmt.Println("Christmas 2024:", t2)

    // 從 Unix 時間戳創建
    timestamp := int64(1704067200) // 2024-01-01 00:00:00 UTC
    t3 := time.Unix(timestamp, 0)
    fmt.Println("From timestamp:", t3)

    // 從毫秒時間戳創建
    timestampMs := int64(1704067200000)
    t4 := time.UnixMilli(timestampMs)
    fmt.Println("From ms timestamp:", t4)

    // 零值時間
    var t5 time.Time
    fmt.Println("Zero time:", t5)
    fmt.Println("Is zero:", t5.IsZero())
}
```

**Node.js:**
```javascript
// 創建特定時間
const t1 = new Date(Date.UTC(2024, 0, 1, 0, 0, 0));
console.log("New Year 2024:", t1);

// 使用本地時區
const t2 = new Date(2024, 11, 25, 12, 30, 0);
console.log("Christmas 2024:", t2);

// 從 Unix 時間戳創建（秒）
const timestamp = 1704067200;
const t3 = new Date(timestamp * 1000);
console.log("From timestamp:", t3);

// 從毫秒時間戳創建
const timestampMs = 1704067200000;
const t4 = new Date(timestampMs);
console.log("From ms timestamp:", t4);

// 無效日期
const t5 = new Date("invalid");
console.log("Invalid date:", t5);
console.log("Is invalid:", isNaN(t5.getTime()));
```

### 示例 3: 時間格式化

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    now := time.Now()

    // 常用格式
    fmt.Println("ISO 8601:", now.Format(time.RFC3339))
    fmt.Println("Date only:", now.Format("2006-01-02"))
    fmt.Println("Time only:", now.Format("15:04:05"))
    fmt.Println("DateTime:", now.Format("2006-01-02 15:04:05"))

    // 自定義格式
    fmt.Println("Custom 1:", now.Format("Jan 2, 2006"))
    fmt.Println("Custom 2:", now.Format("Monday, January 2, 2006"))
    fmt.Println("Custom 3:", now.Format("3:04 PM"))
    fmt.Println("Custom 4:", now.Format("2006/01/02 15:04:05 MST"))

    // 常用預定義格式
    fmt.Println("ANSIC:", now.Format(time.ANSIC))
    fmt.Println("UnixDate:", now.Format(time.UnixDate))
    fmt.Println("RFC822:", now.Format(time.RFC822))
    fmt.Println("RFC3339:", now.Format(time.RFC3339))
    fmt.Println("Kitchen:", now.Format(time.Kitchen))

    // 中文格式
    fmt.Println("中文:", now.Format("2006年01月02日 15:04:05"))
}
```

**Node.js:**
```javascript
const now = new Date();

// 常用格式
console.log("ISO 8601:", now.toISOString());
console.log("Date only:", now.toISOString().split('T')[0]);
console.log("Time only:", now.toTimeString().split(' ')[0]);
console.log("DateTime:", now.toISOString().replace('T', ' ').split('.')[0]);

// 自定義格式（需要手動實現或使用庫）
const months = ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'];
const fullMonths = ['January', 'February', 'March', 'April', 'May', 'June', 'July', 'August', 'September', 'October', 'November', 'December'];
const days = ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'];

console.log("Custom 1:", `${months[now.getMonth()]} ${now.getDate()}, ${now.getFullYear()}`);
console.log("Custom 2:", `${days[now.getDay()]}, ${fullMonths[now.getMonth()]} ${now.getDate()}, ${now.getFullYear()}`);

const hours = now.getHours();
const ampm = hours >= 12 ? 'PM' : 'AM';
const hours12 = hours % 12 || 12;
console.log("Custom 3:", `${hours12}:${String(now.getMinutes()).padStart(2, '0')} ${ampm}`);

// 使用 toLocaleString
console.log("Locale:", now.toLocaleString('en-US'));
console.log("中文:", now.toLocaleString('zh-CN'));
```

### 示例 4: 時間解析

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 解析標準格式
    str1 := "2024-01-15"
    t1, err := time.Parse("2006-01-02", str1)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Parsed:", t1)
    }

    // 解析日期時間
    str2 := "2024-01-15 14:30:00"
    t2, err := time.Parse("2006-01-02 15:04:05", str2)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Parsed:", t2)
    }

    // 解析 RFC3339
    str3 := "2024-01-15T14:30:00Z"
    t3, err := time.Parse(time.RFC3339, str3)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Parsed:", t3)
    }

    // ParseInLocation - 指定時區
    str4 := "2024-01-15 14:30:00"
    loc, _ := time.LoadLocation("Asia/Tokyo")
    t4, err := time.ParseInLocation("2006-01-02 15:04:05", str4, loc)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Parsed (Tokyo):", t4)
    }

    // 錯誤處理
    str5 := "invalid-date"
    _, err = time.Parse("2006-01-02", str5)
    if err != nil {
        fmt.Printf("Parse error: %v\n", err)
    }
}
```

**Node.js:**
```javascript
// 解析標準格式
const str1 = "2024-01-15";
const t1 = new Date(str1);
console.log("Parsed:", t1);

// 解析日期時間
const str2 = "2024-01-15 14:30:00";
const t2 = new Date(str2.replace(' ', 'T'));
console.log("Parsed:", t2);

// 解析 ISO 8601
const str3 = "2024-01-15T14:30:00Z";
const t3 = new Date(str3);
console.log("Parsed:", t3);

// 指定時區（需要處理）
const str4 = "2024-01-15 14:30:00";
const t4 = new Date(str4 + "+09:00"); // Tokyo timezone
console.log("Parsed (Tokyo):", t4);

// 錯誤處理
const str5 = "invalid-date";
const t5 = new Date(str5);
if (isNaN(t5.getTime())) {
    console.log("Parse error: Invalid date");
}
```

### 示例 5: 時間運算

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    now := time.Now()

    // 添加時間
    tomorrow := now.Add(24 * time.Hour)
    fmt.Println("Tomorrow:", tomorrow.Format("2006-01-02"))

    nextWeek := now.Add(7 * 24 * time.Hour)
    fmt.Println("Next week:", nextWeek.Format("2006-01-02"))

    // 減去時間
    yesterday := now.Add(-24 * time.Hour)
    fmt.Println("Yesterday:", yesterday.Format("2006-01-02"))

    // AddDate - 添加年月日
    nextYear := now.AddDate(1, 0, 0)
    fmt.Println("Next year:", nextYear.Format("2006-01-02"))

    nextMonth := now.AddDate(0, 1, 0)
    fmt.Println("Next month:", nextMonth.Format("2006-01-02"))

    nextWeek2 := now.AddDate(0, 0, 7)
    fmt.Println("Next week:", nextWeek2.Format("2006-01-02"))

    // 計算時間差
    past := time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)
    duration := now.Sub(past)

    fmt.Printf("Duration: %v\n", duration)
    fmt.Printf("Hours: %.2f\n", duration.Hours())
    fmt.Printf("Days: %.2f\n", duration.Hours()/24)

    // 時間比較
    t1 := time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC)
    t2 := time.Date(2024, 12, 31, 0, 0, 0, 0, time.UTC)

    fmt.Println("t1 before t2:", t1.Before(t2))
    fmt.Println("t1 after t2:", t1.After(t2))
    fmt.Println("t1 equal t2:", t1.Equal(t2))
}
```

**Node.js:**
```javascript
const now = new Date();

// 添加時間
const tomorrow = new Date(now.getTime() + 24 * 60 * 60 * 1000);
console.log("Tomorrow:", tomorrow.toISOString().split('T')[0]);

const nextWeek = new Date(now.getTime() + 7 * 24 * 60 * 60 * 1000);
console.log("Next week:", nextWeek.toISOString().split('T')[0]);

// 減去時間
const yesterday = new Date(now.getTime() - 24 * 60 * 60 * 1000);
console.log("Yesterday:", yesterday.toISOString().split('T')[0]);

// 添加年月日
const nextYear = new Date(now);
nextYear.setFullYear(now.getFullYear() + 1);
console.log("Next year:", nextYear.toISOString().split('T')[0]);

const nextMonth = new Date(now);
nextMonth.setMonth(now.getMonth() + 1);
console.log("Next month:", nextMonth.toISOString().split('T')[0]);

// 計算時間差
const past = new Date('2024-01-01T00:00:00Z');
const duration = now - past; // 毫秒

console.log(`Duration: ${duration}ms`);
console.log(`Hours: ${(duration / (1000 * 60 * 60)).toFixed(2)}`);
console.log(`Days: ${(duration / (1000 * 60 * 60 * 24)).toFixed(2)}`);

// 時間比較
const t1 = new Date('2024-01-01T00:00:00Z');
const t2 = new Date('2024-12-31T00:00:00Z');

console.log("t1 before t2:", t1 < t2);
console.log("t1 after t2:", t1 > t2);
console.log("t1 equal t2:", t1.getTime() === t2.getTime());
```

### 示例 6: Duration 處理

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 創建 Duration
    d1 := 5 * time.Second
    d2 := 3 * time.Minute
    d3 := 2 * time.Hour

    fmt.Printf("5 seconds: %v\n", d1)
    fmt.Printf("3 minutes: %v\n", d2)
    fmt.Printf("2 hours: %v\n", d3)

    // Duration 運算
    total := d1 + d2 + d3
    fmt.Printf("Total: %v\n", total)

    // 轉換為不同單位
    d := 90 * time.Minute
    fmt.Printf("Duration: %v\n", d)
    fmt.Printf("Seconds: %f\n", d.Seconds())
    fmt.Printf("Minutes: %f\n", d.Minutes())
    fmt.Printf("Hours: %f\n", d.Hours())

    // 解析 Duration 字符串
    d4, _ := time.ParseDuration("1h30m")
    fmt.Printf("Parsed: %v\n", d4)

    d5, _ := time.ParseDuration("2h45m30s")
    fmt.Printf("Parsed: %v\n", d5)

    // 截斷 Duration
    d6 := 125 * time.Millisecond
    fmt.Printf("Original: %v\n", d6)
    fmt.Printf("Truncated to 10ms: %v\n", d6.Truncate(10*time.Millisecond))
    fmt.Printf("Rounded to 10ms: %v\n", d6.Round(10*time.Millisecond))
}
```

**Node.js:**
```javascript
// 創建時間間隔（毫秒）
const d1 = 5 * 1000;            // 5 秒
const d2 = 3 * 60 * 1000;       // 3 分鐘
const d3 = 2 * 60 * 60 * 1000;  // 2 小時

console.log(`5 seconds: ${d1}ms`);
console.log(`3 minutes: ${d2}ms`);
console.log(`2 hours: ${d3}ms`);

// Duration 運算
const total = d1 + d2 + d3;
console.log(`Total: ${total}ms`);

// 轉換為不同單位
const d = 90 * 60 * 1000; // 90 分鐘
console.log(`Duration: ${d}ms`);
console.log(`Seconds: ${d / 1000}`);
console.log(`Minutes: ${d / (1000 * 60)}`);
console.log(`Hours: ${d / (1000 * 60 * 60)}`);

// 解析 Duration 字符串（需要自定義）
function parseDuration(s) {
    const regex = /(\d+)h|(\d+)m|(\d+)s/g;
    let ms = 0;
    let match;
    while ((match = regex.exec(s)) !== null) {
        if (match[1]) ms += parseInt(match[1]) * 60 * 60 * 1000; // hours
        if (match[2]) ms += parseInt(match[2]) * 60 * 1000;      // minutes
        if (match[3]) ms += parseInt(match[3]) * 1000;           // seconds
    }
    return ms;
}

console.log(`Parsed: ${parseDuration("1h30m")}ms`);
console.log(`Parsed: ${parseDuration("2h45m30s")}ms`);
```

### 示例 7: 定時器和睡眠

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Sleep - 阻塞當前 goroutine
    fmt.Println("Start")
    time.Sleep(2 * time.Second)
    fmt.Println("After 2 seconds")

    // After - 返回 channel，延遲後發送時間
    fmt.Println("Waiting...")
    <-time.After(1 * time.Second)
    fmt.Println("Done!")

    // Tick - 定時發送信號
    ticker := time.NewTicker(500 * time.Millisecond)
    count := 0
    for range ticker.C {
        count++
        fmt.Printf("Tick %d\n", count)
        if count >= 5 {
            ticker.Stop()
            break
        }
    }

    // Timer - 單次定時器
    timer := time.NewTimer(2 * time.Second)
    fmt.Println("Timer started")
    <-timer.C
    fmt.Println("Timer fired!")

    // AfterFunc - 延遲執行函數
    time.AfterFunc(1*time.Second, func() {
        fmt.Println("Executed after 1 second")
    })
    time.Sleep(2 * time.Second) // 等待 AfterFunc 執行
}
```

**Node.js:**
```javascript
// Sleep 函數（Promise 版本）
const sleep = (ms) => new Promise(resolve => setTimeout(resolve, ms));

async function main() {
    // Sleep
    console.log("Start");
    await sleep(2000);
    console.log("After 2 seconds");

    // After（類似）
    console.log("Waiting...");
    await sleep(1000);
    console.log("Done!");

    // Interval - 定時執行
    let count = 0;
    const interval = setInterval(() => {
        count++;
        console.log(`Tick ${count}`);
        if (count >= 5) {
            clearInterval(interval);
        }
    }, 500);

    await sleep(3000); // 等待 interval 完成

    // Timer - 單次定時器
    console.log("Timer started");
    await new Promise(resolve => {
        setTimeout(() => {
            console.log("Timer fired!");
            resolve();
        }, 2000);
    });

    // AfterFunc（類似）
    setTimeout(() => {
        console.log("Executed after 1 second");
    }, 1000);
    await sleep(2000); // 等待執行
}

main();
```

### 示例 8: 時區處理

**Go:**
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    now := time.Now()

    // 獲取當前時區
    fmt.Println("Local time:", now)
    fmt.Println("Timezone:", now.Location())

    // UTC 時間
    utc := now.UTC()
    fmt.Println("UTC time:", utc)

    // 加載特定時區
    tokyo, _ := time.LoadLocation("Asia/Tokyo")
    tokyoTime := now.In(tokyo)
    fmt.Println("Tokyo time:", tokyoTime)

    newYork, _ := time.LoadLocation("America/New_York")
    nyTime := now.In(newYork)
    fmt.Println("New York time:", nyTime)

    // 創建指定時區的時間
    t := time.Date(2024, 1, 1, 12, 0, 0, 0, tokyo)
    fmt.Println("Specific timezone:", t)

    // 時區轉換
    fmt.Println("Tokyo -> UTC:", t.UTC())
    fmt.Println("Tokyo -> NY:", t.In(newYork))
}
```

**Node.js:**
```javascript
const now = new Date();

// 獲取當前時區
console.log("Local time:", now);
console.log("Timezone offset:", now.getTimezoneOffset(), "minutes");

// UTC 時間
const utc = new Date(now.toUTCString());
console.log("UTC time:", utc.toISOString());

// 不同時區的時間（使用 Intl）
console.log("Tokyo time:", now.toLocaleString('en-US', { timeZone: 'Asia/Tokyo' }));
console.log("New York time:", now.toLocaleString('en-US', { timeZone: 'America/New_York' }));

// 或使用第三方庫（如 moment-timezone）
// const moment = require('moment-timezone');
// console.log("Tokyo:", moment().tz("Asia/Tokyo").format());
// console.log("New York:", moment().tz("America/New_York").format());
```

## 重點總結

1. **Time 類型**
   - `time.Now()` - 獲取當前時間
   - `time.Date()` - 創建特定時間
   - 納秒級精度，包含時區信息

2. **Duration 類型**
   - 表示時間間隔，單位為納秒
   - 支持運算：加減乘除
   - 常用單位：Second, Minute, Hour

3. **格式化和解析**
   - `Format()` - 格式化時間為字符串
   - `Parse()` - 解析字符串為時間
   - 使用參考時間 "2006-01-02 15:04:05" 作為布局

4. **時間運算**
   - `Add()` - 添加 Duration
   - `AddDate()` - 添加年月日
   - `Sub()` - 計算時間差
   - `Before()`, `After()`, `Equal()` - 時間比較

5. **定時器**
   - `Sleep()` - 睡眠
   - `After()` - 延遲 channel
   - `Tick()` - 定時 channel
   - `Timer` - 單次定時器
   - `Ticker` - 週期定時器

6. **與 Node.js 的主要差異**
   - Go 的時間格式化更直觀（使用參考時間）
   - Go 的 Duration 類型更明確
   - Go 的時區處理更完善
   - Go 的定時器基於 channel

## 練習題

### 練習 1: 年齡計算器
編寫函數計算兩個日期之間的年齡（年、月、日）。

### 練習 2: 工作日計算器
編寫函數計算兩個日期之間的工作日數量（排除週末）。

### 練習 3: 倒計時
創建倒計時程序，顯示距離特定日期還有多少天、小時、分鐘、秒。

### 練習 4: 時間格式化
實現一個函數，將 Duration 格式化為人類可讀的字符串（如 "2h 30m 45s"）。

### 練習 5: 定時任務
創建一個定時任務調度器，每天特定時間執行任務。

### 練習 6: 時區轉換工具
創建時區轉換工具，支持多個城市之間的時間轉換。

### 練習 7: 時間範圍檢查
編寫函數檢查時間是否在指定範圍內。

### 練習 8: 性能測試
使用 time 包測量函數執行時間。

## 參考答案

### 練習 1 答案
```go
package main

import (
    "fmt"
    "time"
)

type Age struct {
    Years  int
    Months int
    Days   int
}

func calculateAge(birthdate, now time.Time) Age {
    years := now.Year() - birthdate.Year()
    months := int(now.Month()) - int(birthdate.Month())
    days := now.Day() - birthdate.Day()

    if days < 0 {
        months--
        // 獲取上個月的天數
        prevMonth := now.AddDate(0, -1, 0)
        days += daysInMonth(prevMonth)
    }

    if months < 0 {
        years--
        months += 12
    }

    return Age{Years: years, Months: months, Days: days}
}

func daysInMonth(t time.Time) int {
    return time.Date(t.Year(), t.Month()+1, 0, 0, 0, 0, 0, t.Location()).Day()
}

func main() {
    birthdate := time.Date(1990, 5, 15, 0, 0, 0, 0, time.UTC)
    now := time.Now()

    age := calculateAge(birthdate, now)
    fmt.Printf("Age: %d years, %d months, %d days\n", age.Years, age.Months, age.Days)
}
```

### 練習 3 答案
```go
package main

import (
    "fmt"
    "time"
)

func countdown(target time.Time) {
    for {
        now := time.Now()
        duration := target.Sub(now)

        if duration <= 0 {
            fmt.Println("Time's up!")
            break
        }

        days := int(duration.Hours() / 24)
        hours := int(duration.Hours()) % 24
        minutes := int(duration.Minutes()) % 60
        seconds := int(duration.Seconds()) % 60

        fmt.Printf("\rCountdown: %d days, %d hours, %d minutes, %d seconds",
            days, hours, minutes, seconds)

        time.Sleep(1 * time.Second)
    }
}

func main() {
    target := time.Date(2024, 12, 31, 23, 59, 59, 0, time.Local)
    countdown(target)
}
```

### 練習 8 答案
```go
package main

import (
    "fmt"
    "time"
)

func measureTime(fn func()) time.Duration {
    start := time.Now()
    fn()
    return time.Since(start)
}

func slowFunction() {
    time.Sleep(500 * time.Millisecond)
    // 模擬耗時操作
    sum := 0
    for i := 0; i < 1000000; i++ {
        sum += i
    }
}

func main() {
    duration := measureTime(slowFunction)
    fmt.Printf("Function took: %v\n", duration)
    fmt.Printf("Function took: %.2f seconds\n", duration.Seconds())
}
```
