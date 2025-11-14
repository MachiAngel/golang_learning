# Chapter 32: io 與文件操作

## 概述

Go 的 `io` 和 `os` 包提供了強大的文件和 I/O 操作功能。對於 Node.js 開發者來說，它們相當於 `fs` 模塊，但提供了更靈活的接口設計和更好的性能。

## Node.js 與 Go 的對比

| 功能 | Node.js | Go |
|------|---------|-----|
| 讀取文件 | `fs.readFileSync(path)` | `os.ReadFile(path)` |
| 寫入文件 | `fs.writeFileSync(path, data)` | `os.WriteFile(path, data, perm)` |
| 追加文件 | `fs.appendFileSync(path, data)` | `os.OpenFile()` + `Write()` |
| 檢查文件存在 | `fs.existsSync(path)` | `os.Stat(path)` |
| 刪除文件 | `fs.unlinkSync(path)` | `os.Remove(path)` |
| 創建目錄 | `fs.mkdirSync(path)` | `os.Mkdir(path, perm)` |
| 讀取目錄 | `fs.readdirSync(path)` | `os.ReadDir(path)` |
| 複製文件 | `fs.copyFileSync(src, dst)` | `io.Copy()` |
| 流式讀取 | `fs.createReadStream()` | `os.Open()` + `bufio.Reader` |
| 流式寫入 | `fs.createWriteStream()` | `os.Create()` + `bufio.Writer` |

## 詳細概念解釋

### 1. io 包接口

Go 的 I/O 操作基於接口：

- `io.Reader` - 讀取數據的接口
- `io.Writer` - 寫入數據的接口
- `io.Closer` - 關閉資源的接口
- `io.ReadWriter` - Reader + Writer
- `io.ReadCloser` - Reader + Closer
- `io.WriteCloser` - Writer + Closer

### 2. 常用函數

**io 包**：
- `io.Copy(dst, src)` - 複製數據
- `io.ReadAll(r)` - 讀取所有數據
- `io.WriteString(w, s)` - 寫入字符串

**os 包**：
- `os.ReadFile(name)` - 讀取整個文件
- `os.WriteFile(name, data, perm)` - 寫入整個文件
- `os.Open(name)` - 打開文件（只讀）
- `os.Create(name)` - 創建文件（覆蓋）
- `os.OpenFile(name, flag, perm)` - 以指定模式打開文件

### 3. 文件權限

Go 使用 Unix 風格的文件權限（八進制）：
- `0644` - 所有者可讀寫，其他人只讀
- `0755` - 所有者可讀寫執行，其他人可讀執行
- `0666` - 所有人可讀寫

### 4. 文件打開標誌

- `os.O_RDONLY` - 只讀
- `os.O_WRONLY` - 只寫
- `os.O_RDWR` - 讀寫
- `os.O_CREATE` - 如果不存在則創建
- `os.O_APPEND` - 追加模式
- `os.O_TRUNC` - 打開時清空

## 實際代碼示例

### 示例 1: 讀取和寫入文件

**Go:**
```go
package main

import (
    "fmt"
    "os"
)

func main() {
    filename := "test.txt"

    // 寫入文件
    content := []byte("Hello, Go!\nThis is a test file.")
    err := os.WriteFile(filename, content, 0644)
    if err != nil {
        fmt.Println("Error writing file:", err)
        return
    }
    fmt.Println("File written successfully")

    // 讀取文件
    data, err := os.ReadFile(filename)
    if err != nil {
        fmt.Println("Error reading file:", err)
        return
    }
    fmt.Println("File content:")
    fmt.Println(string(data))

    // 清理
    os.Remove(filename)
}
```

**Node.js:**
```javascript
const fs = require('fs');

const filename = 'test.txt';

// 寫入文件
const content = "Hello, Node.js!\nThis is a test file.";
try {
    fs.writeFileSync(filename, content);
    console.log("File written successfully");
} catch (err) {
    console.error("Error writing file:", err.message);
}

// 讀取文件
try {
    const data = fs.readFileSync(filename, 'utf8');
    console.log("File content:");
    console.log(data);
} catch (err) {
    console.error("Error reading file:", err.message);
}

// 清理
fs.unlinkSync(filename);
```

### 示例 2: 追加文件和不同的打開模式

**Go:**
```go
package main

import (
    "fmt"
    "os"
)

func main() {
    filename := "log.txt"

    // 創建並寫入
    file, err := os.Create(filename)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    file.WriteString("Line 1\n")
    file.Close()

    // 追加模式打開
    file, err = os.OpenFile(filename, os.O_APPEND|os.O_WRONLY, 0644)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer file.Close()

    file.WriteString("Line 2\n")
    file.WriteString("Line 3\n")

    // 讀取結果
    data, _ := os.ReadFile(filename)
    fmt.Println("File content:")
    fmt.Println(string(data))

    // 清理
    os.Remove(filename)
}
```

**Node.js:**
```javascript
const fs = require('fs');

const filename = 'log.txt';

// 創建並寫入
fs.writeFileSync(filename, "Line 1\n");

// 追加
fs.appendFileSync(filename, "Line 2\n");
fs.appendFileSync(filename, "Line 3\n");

// 讀取結果
const data = fs.readFileSync(filename, 'utf8');
console.log("File content:");
console.log(data);

// 清理
fs.unlinkSync(filename);
```

### 示例 3: 逐行讀取文件

**Go:**
```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    filename := "lines.txt"

    // 創建測試文件
    os.WriteFile(filename, []byte("Line 1\nLine 2\nLine 3\nLine 4\nLine 5\n"), 0644)

    // 打開文件
    file, err := os.Open(filename)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer file.Close()

    // 使用 Scanner 逐行讀取
    scanner := bufio.NewScanner(file)
    lineNum := 1

    for scanner.Scan() {
        line := scanner.Text()
        fmt.Printf("%d: %s\n", lineNum, line)
        lineNum++
    }

    if err := scanner.Err(); err != nil {
        fmt.Println("Error reading file:", err)
    }

    // 清理
    os.Remove(filename)
}
```

**Node.js:**
```javascript
const fs = require('fs');
const readline = require('readline');

const filename = 'lines.txt';

// 創建測試文件
fs.writeFileSync(filename, "Line 1\nLine 2\nLine 3\nLine 4\nLine 5\n");

// 逐行讀取
const fileStream = fs.createReadStream(filename);
const rl = readline.createInterface({
    input: fileStream,
    crlfDelay: Infinity
});

let lineNum = 1;
rl.on('line', (line) => {
    console.log(`${lineNum}: ${line}`);
    lineNum++;
});

rl.on('close', () => {
    // 清理
    fs.unlinkSync(filename);
});
```

### 示例 4: 緩衝讀寫

**Go:**
```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    filename := "buffered.txt"

    // 緩衝寫入
    file, err := os.Create(filename)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }

    writer := bufio.NewWriter(file)
    for i := 1; i <= 5; i++ {
        writer.WriteString(fmt.Sprintf("Line %d\n", i))
    }
    writer.Flush() // 刷新緩衝區
    file.Close()

    // 緩衝讀取
    file, err = os.Open(filename)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer file.Close()

    reader := bufio.NewReader(file)
    fmt.Println("File content:")

    for {
        line, err := reader.ReadString('\n')
        if err != nil {
            break
        }
        fmt.Print(line)
    }

    // 清理
    os.Remove(filename)
}
```

**Node.js:**
```javascript
const fs = require('fs');

const filename = 'buffered.txt';

// 緩衝寫入
const writeStream = fs.createWriteStream(filename);
for (let i = 1; i <= 5; i++) {
    writeStream.write(`Line ${i}\n`);
}
writeStream.end();

writeStream.on('finish', () => {
    // 緩衝讀取
    const readStream = fs.createReadStream(filename, {
        encoding: 'utf8',
        highWaterMark: 16 * 1024 // 16KB buffer
    });

    console.log("File content:");
    readStream.on('data', (chunk) => {
        process.stdout.write(chunk);
    });

    readStream.on('end', () => {
        // 清理
        fs.unlinkSync(filename);
    });
});
```

### 示例 5: 複製文件

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "os"
)

func copyFile(src, dst string) error {
    // 打開源文件
    sourceFile, err := os.Open(src)
    if err != nil {
        return err
    }
    defer sourceFile.Close()

    // 創建目標文件
    destFile, err := os.Create(dst)
    if err != nil {
        return err
    }
    defer destFile.Close()

    // 複製內容
    bytesWritten, err := io.Copy(destFile, sourceFile)
    if err != nil {
        return err
    }

    fmt.Printf("Copied %d bytes\n", bytesWritten)

    // 同步到磁盤
    err = destFile.Sync()
    return err
}

func main() {
    // 創建源文件
    src := "source.txt"
    os.WriteFile(src, []byte("This is the source file content."), 0644)

    // 複製文件
    dst := "destination.txt"
    err := copyFile(src, dst)
    if err != nil {
        fmt.Println("Error copying file:", err)
        return
    }

    // 驗證
    data, _ := os.ReadFile(dst)
    fmt.Println("Destination file content:", string(data))

    // 清理
    os.Remove(src)
    os.Remove(dst)
}
```

**Node.js:**
```javascript
const fs = require('fs');

function copyFile(src, dst) {
    try {
        fs.copyFileSync(src, dst);
        const stats = fs.statSync(dst);
        console.log(`Copied ${stats.size} bytes`);
    } catch (err) {
        throw err;
    }
}

// 創建源文件
const src = 'source.txt';
fs.writeFileSync(src, 'This is the source file content.');

// 複製文件
const dst = 'destination.txt';
try {
    copyFile(src, dst);

    // 驗證
    const data = fs.readFileSync(dst, 'utf8');
    console.log("Destination file content:", data);
} catch (err) {
    console.error("Error copying file:", err.message);
}

// 清理
fs.unlinkSync(src);
fs.unlinkSync(dst);
```

### 示例 6: 文件信息和元數據

**Go:**
```go
package main

import (
    "fmt"
    "os"
    "time"
)

func main() {
    filename := "info.txt"

    // 創建文件
    os.WriteFile(filename, []byte("Test file content"), 0644)

    // 獲取文件信息
    fileInfo, err := os.Stat(filename)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }

    fmt.Println("File name:", fileInfo.Name())
    fmt.Println("Size:", fileInfo.Size(), "bytes")
    fmt.Println("Permissions:", fileInfo.Mode())
    fmt.Println("Modified time:", fileInfo.ModTime())
    fmt.Println("Is directory:", fileInfo.IsDir())

    // 檢查文件是否存在
    if _, err := os.Stat("nonexistent.txt"); os.IsNotExist(err) {
        fmt.Println("nonexistent.txt does not exist")
    }

    // 修改文件權限
    err = os.Chmod(filename, 0755)
    if err != nil {
        fmt.Println("Error changing permissions:", err)
    }

    // 修改文件時間
    newTime := time.Now().Add(-24 * time.Hour)
    err = os.Chtimes(filename, newTime, newTime)
    if err != nil {
        fmt.Println("Error changing time:", err)
    }

    // 驗證修改
    fileInfo, _ = os.Stat(filename)
    fmt.Println("New modified time:", fileInfo.ModTime())
    fmt.Println("New permissions:", fileInfo.Mode())

    // 清理
    os.Remove(filename)
}
```

**Node.js:**
```javascript
const fs = require('fs');

const filename = 'info.txt';

// 創建文件
fs.writeFileSync(filename, 'Test file content');

// 獲取文件信息
const stats = fs.statSync(filename);

console.log("File name:", filename);
console.log("Size:", stats.size, "bytes");
console.log("Permissions:", stats.mode.toString(8));
console.log("Modified time:", stats.mtime);
console.log("Is directory:", stats.isDirectory());

// 檢查文件是否存在
if (!fs.existsSync('nonexistent.txt')) {
    console.log("nonexistent.txt does not exist");
}

// 修改文件權限
fs.chmodSync(filename, 0o755);

// 修改文件時間
const newTime = new Date(Date.now() - 24 * 60 * 60 * 1000);
fs.utimesSync(filename, newTime, newTime);

// 驗證修改
const newStats = fs.statSync(filename);
console.log("New modified time:", newStats.mtime);
console.log("New permissions:", newStats.mode.toString(8));

// 清理
fs.unlinkSync(filename);
```

### 示例 7: 目錄操作

**Go:**
```go
package main

import (
    "fmt"
    "os"
    "path/filepath"
)

func main() {
    // 創建目錄
    err := os.Mkdir("testdir", 0755)
    if err != nil {
        fmt.Println("Error creating directory:", err)
        return
    }

    // 創建嵌套目錄
    err = os.MkdirAll("parent/child/grandchild", 0755)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }

    // 在目錄中創建文件
    os.WriteFile("testdir/file1.txt", []byte("File 1"), 0644)
    os.WriteFile("testdir/file2.txt", []byte("File 2"), 0644)

    // 讀取目錄內容
    entries, err := os.ReadDir("testdir")
    if err != nil {
        fmt.Println("Error reading directory:", err)
        return
    }

    fmt.Println("Directory contents:")
    for _, entry := range entries {
        fmt.Printf("  %s (dir: %v)\n", entry.Name(), entry.IsDir())
    }

    // 遍歷目錄樹
    fmt.Println("\nWalking directory tree:")
    filepath.Walk("parent", func(path string, info os.FileInfo, err error) error {
        if err != nil {
            return err
        }
        fmt.Println(path)
        return nil
    })

    // 刪除目錄（必須為空）
    os.Remove("testdir/file1.txt")
    os.Remove("testdir/file2.txt")
    os.Remove("testdir")

    // 遞歸刪除目錄
    os.RemoveAll("parent")
}
```

**Node.js:**
```javascript
const fs = require('fs');
const path = require('path');

// 創建目錄
fs.mkdirSync('testdir');

// 創建嵌套目錄
fs.mkdirSync('parent/child/grandchild', { recursive: true });

// 在目錄中創建文件
fs.writeFileSync('testdir/file1.txt', 'File 1');
fs.writeFileSync('testdir/file2.txt', 'File 2');

// 讀取目錄內容
const entries = fs.readdirSync('testdir', { withFileTypes: true });

console.log("Directory contents:");
entries.forEach(entry => {
    console.log(`  ${entry.name} (dir: ${entry.isDirectory()})`);
});

// 遍歷目錄樹
function walkDir(dir) {
    const entries = fs.readdirSync(dir, { withFileTypes: true });
    entries.forEach(entry => {
        const fullPath = path.join(dir, entry.name);
        console.log(fullPath);
        if (entry.isDirectory()) {
            walkDir(fullPath);
        }
    });
}

console.log("\nWalking directory tree:");
console.log('parent');
walkDir('parent');

// 刪除目錄（必須為空）
fs.unlinkSync('testdir/file1.txt');
fs.unlinkSync('testdir/file2.txt');
fs.rmdirSync('testdir');

// 遞歸刪除目錄
fs.rmSync('parent', { recursive: true });
```

### 示例 8: 臨時文件和目錄

**Go:**
```go
package main

import (
    "fmt"
    "os"
)

func main() {
    // 創建臨時文件
    tmpFile, err := os.CreateTemp("", "example-*.txt")
    if err != nil {
        fmt.Println("Error creating temp file:", err)
        return
    }
    defer os.Remove(tmpFile.Name())

    fmt.Println("Temp file:", tmpFile.Name())

    // 寫入臨時文件
    tmpFile.WriteString("This is temporary data")
    tmpFile.Close()

    // 讀取臨時文件
    data, _ := os.ReadFile(tmpFile.Name())
    fmt.Println("Content:", string(data))

    // 創建臨時目錄
    tmpDir, err := os.MkdirTemp("", "example-dir-")
    if err != nil {
        fmt.Println("Error creating temp dir:", err)
        return
    }
    defer os.RemoveAll(tmpDir)

    fmt.Println("Temp directory:", tmpDir)

    // 在臨時目錄中創建文件
    tmpFilePath := tmpDir + "/data.txt"
    os.WriteFile(tmpFilePath, []byte("Temp directory file"), 0644)

    // 驗證
    files, _ := os.ReadDir(tmpDir)
    fmt.Println("Files in temp dir:")
    for _, file := range files {
        fmt.Println("  ", file.Name())
    }
}
```

**Node.js:**
```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');

// 創建臨時文件
const tmpDir = os.tmpdir();
const tmpFile = path.join(tmpDir, `example-${Date.now()}.txt`);

console.log("Temp file:", tmpFile);

// 寫入臨時文件
fs.writeFileSync(tmpFile, 'This is temporary data');

// 讀取臨時文件
const data = fs.readFileSync(tmpFile, 'utf8');
console.log("Content:", data);

// 清理臨時文件
fs.unlinkSync(tmpFile);

// 創建臨時目錄
const tmpDirPath = fs.mkdtempSync(path.join(tmpDir, 'example-dir-'));
console.log("Temp directory:", tmpDirPath);

// 在臨時目錄中創建文件
const tmpFilePath = path.join(tmpDirPath, 'data.txt');
fs.writeFileSync(tmpFilePath, 'Temp directory file');

// 驗證
const files = fs.readdirSync(tmpDirPath);
console.log("Files in temp dir:");
files.forEach(file => {
    console.log("  ", file);
});

// 清理臨時目錄
fs.rmSync(tmpDirPath, { recursive: true });
```

## 重點總結

1. **基本文件操作**
   - `os.ReadFile()` / `os.WriteFile()` - 簡單讀寫
   - `os.Open()` / `os.Create()` - 打開/創建文件
   - `os.Remove()` / `os.RemoveAll()` - 刪除文件/目錄

2. **緩衝 I/O**
   - `bufio.Reader` - 緩衝讀取
   - `bufio.Writer` - 緩衝寫入
   - `bufio.Scanner` - 逐行掃描

3. **io 包接口**
   - `io.Reader` / `io.Writer` - 核心接口
   - `io.Copy()` - 數據複製
   - `io.ReadAll()` - 讀取所有數據

4. **目錄操作**
   - `os.Mkdir()` / `os.MkdirAll()` - 創建目錄
   - `os.ReadDir()` - 讀取目錄
   - `filepath.Walk()` - 遍歷目錄樹

5. **與 Node.js 的主要差異**
   - Go 需要顯式關閉文件（defer）
   - Go 使用 Unix 風格的權限
   - Go 的 I/O 基於接口，更靈活
   - Go 的錯誤處理更明確

6. **最佳實踐**
   - 總是使用 defer file.Close()
   - 檢查所有錯誤
   - 使用緩衝 I/O 處理大文件
   - 使用 os.ReadFile() 處理小文件
   - 使用 filepath 包處理路徑

## 練習題

### 練習 1: 文件統計
編寫程序統計文件的行數、單詞數和字符數。

### 練習 2: 文件搜索
實現在目錄樹中搜索包含特定文本的文件。

### 練習 3: 日誌輪轉
實現簡單的日誌輪轉功能（當文件大小超過限制時創建新文件）。

### 練習 4: 配置文件管理
創建配置文件讀寫工具，支持 JSON 和 YAML 格式。

### 練習 5: 文件監控
實現文件變化監控（可使用 fsnotify 包）。

### 練習 6: 批量重命名
編寫批量重命名文件的工具。

### 練習 7: 文件壓縮
實現文件壓縮和解壓縮功能。

### 練習 8: 文件同步
實現簡單的文件同步工具（比較並複製更新的文件）。

## 參考答案

### 練習 1 答案
```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
)

type FileStats struct {
    Lines      int
    Words      int
    Characters int
}

func analyzeFile(filename string) (FileStats, error) {
    file, err := os.Open(filename)
    if err != nil {
        return FileStats{}, err
    }
    defer file.Close()

    stats := FileStats{}
    scanner := bufio.NewScanner(file)

    for scanner.Scan() {
        line := scanner.Text()
        stats.Lines++
        stats.Characters += len(line) + 1 // +1 for newline

        words := strings.Fields(line)
        stats.Words += len(words)
    }

    if err := scanner.Err(); err != nil {
        return FileStats{}, err
    }

    return stats, nil
}

func main() {
    // 創建測試文件
    testFile := "test.txt"
    content := "Hello World\nThis is a test\nGo programming"
    os.WriteFile(testFile, []byte(content), 0644)

    // 分析文件
    stats, err := analyzeFile(testFile)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }

    fmt.Printf("Lines: %d\n", stats.Lines)
    fmt.Printf("Words: %d\n", stats.Words)
    fmt.Printf("Characters: %d\n", stats.Characters)

    // 清理
    os.Remove(testFile)
}
```
