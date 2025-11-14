# Chapter 31: http 客戶端

## 概述

`net/http` 包不僅提供 HTTP 服務器功能，還包含了強大的 HTTP 客戶端。對於 Node.js 開發者來說，它相當於 axios、fetch API 或 node-fetch，但是內置在標準庫中，無需安裝第三方依賴。

## Node.js 與 Go 的對比

| 功能 | Node.js (axios/fetch) | Go |
|------|----------------------|-----|
| GET 請求 | `axios.get(url)` | `http.Get(url)` |
| POST 請求 | `axios.post(url, data)` | `http.Post(url, contentType, body)` |
| 自定義請求 | `axios({method, url, ...})` | `http.NewRequest(method, url, body)` |
| 設置 header | `config.headers = {...}` | `req.Header.Set(key, value)` |
| 超時設置 | `config.timeout = 5000` | `client.Timeout = 5*time.Second` |
| JSON 請求 | `axios.post(url, {...})` | Marshal + Post |
| 響應處理 | `response.data` | `ioutil.ReadAll(resp.Body)` |
| 錯誤處理 | `try/catch` | `if err != nil` |

## 詳細概念解釋

### 1. 基本函數

**便捷函數**（使用默認客戶端）：
- `http.Get(url)` - GET 請求
- `http.Post(url, contentType, body)` - POST 請求
- `http.PostForm(url, data)` - POST 表單
- `http.Head(url)` - HEAD 請求

**自定義請求**：
- `http.NewRequest(method, url, body)` - 創建請求
- `client.Do(req)` - 執行請求

### 2. Client 和 Transport

- `http.Client` - HTTP 客戶端，可配置超時、重定向等
- `http.Transport` - 底層傳輸層，可配置連接池、代理等

### 3. Response 結構

```go
type Response struct {
    Status     string // "200 OK"
    StatusCode int    // 200
    Header     Header // 響應頭
    Body       io.ReadCloser // 響應體
    ...
}
```

## 實際代碼示例

### 示例 1: GET 請求

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "net/http"
)

func main() {
    // 簡單 GET 請求
    resp, err := http.Get("https://api.github.com/users/golang")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body.Close()

    // 檢查狀態碼
    fmt.Println("Status:", resp.Status)
    fmt.Println("Status Code:", resp.StatusCode)

    // 讀取響應頭
    fmt.Println("Content-Type:", resp.Header.Get("Content-Type"))

    // 讀取響應體
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        fmt.Println("Error reading body:", err)
        return
    }

    fmt.Println("Body length:", len(body))
    fmt.Println("Response:", string(body))
}
```

**Node.js (axios):**
```javascript
const axios = require('axios');

async function main() {
    try {
        // 簡單 GET 請求
        const response = await axios.get('https://api.github.com/users/golang');

        // 檢查狀態碼
        console.log("Status:", response.status, response.statusText);

        // 讀取響應頭
        console.log("Content-Type:", response.headers['content-type']);

        // 響應體
        console.log("Body length:", JSON.stringify(response.data).length);
        console.log("Response:", response.data);
    } catch (error) {
        console.error("Error:", error.message);
    }
}

main();
```

**Node.js (fetch):**
```javascript
async function main() {
    try {
        // 簡單 GET 請求
        const response = await fetch('https://api.github.com/users/golang');

        // 檢查狀態碼
        console.log("Status:", response.status, response.statusText);

        // 讀取響應頭
        console.log("Content-Type:", response.headers.get('content-type'));

        // 讀取響應體
        const data = await response.json();
        console.log("Response:", data);
    } catch (error) {
        console.error("Error:", error.message);
    }
}

main();
```

### 示例 2: POST 請求（JSON）

**Go:**
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
)

type User struct {
    Name  string `json:"name"`
    Email string `json:"email"`
}

type Response struct {
    ID      int    `json:"id"`
    Name    string `json:"name"`
    Email   string `json:"email"`
    Message string `json:"message"`
}

func main() {
    // 準備 JSON 數據
    user := User{
        Name:  "Alice",
        Email: "alice@example.com",
    }

    jsonData, err := json.Marshal(user)
    if err != nil {
        fmt.Println("Error marshaling:", err)
        return
    }

    // POST 請求
    resp, err := http.Post(
        "https://jsonplaceholder.typicode.com/users",
        "application/json",
        bytes.NewBuffer(jsonData),
    )
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body.Close()

    // 讀取響應
    body, _ := io.ReadAll(resp.Body)

    // 解析 JSON 響應
    var result Response
    err = json.Unmarshal(body, &result)
    if err != nil {
        fmt.Println("Error unmarshaling:", err)
        return
    }

    fmt.Printf("Response: %+v\n", result)
}
```

**Node.js (axios):**
```javascript
const axios = require('axios');

async function main() {
    try {
        // 準備數據
        const user = {
            name: "Alice",
            email: "alice@example.com"
        };

        // POST 請求
        const response = await axios.post(
            'https://jsonplaceholder.typicode.com/users',
            user
        );

        console.log("Response:", response.data);
    } catch (error) {
        console.error("Error:", error.message);
    }
}

main();
```

### 示例 3: 自定義請求（設置 Headers）

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "net/http"
)

func main() {
    // 創建請求
    req, err := http.NewRequest("GET", "https://api.github.com/user", nil)
    if err != nil {
        fmt.Println("Error creating request:", err)
        return
    }

    // 設置 Headers
    req.Header.Set("Authorization", "Bearer YOUR_TOKEN")
    req.Header.Set("Accept", "application/json")
    req.Header.Set("User-Agent", "My-Go-App/1.0")

    // 創建客戶端並發送請求
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body.Close()

    // 讀取響應
    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Status:", resp.StatusCode)
    fmt.Println("Response:", string(body))
}
```

**Node.js (axios):**
```javascript
const axios = require('axios');

async function main() {
    try {
        // 自定義請求
        const response = await axios({
            method: 'GET',
            url: 'https://api.github.com/user',
            headers: {
                'Authorization': 'Bearer YOUR_TOKEN',
                'Accept': 'application/json',
                'User-Agent': 'My-Node-App/1.0'
            }
        });

        console.log("Status:", response.status);
        console.log("Response:", response.data);
    } catch (error) {
        console.error("Error:", error.message);
    }
}

main();
```

### 示例 4: 查詢參數和 URL 構建

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "net/url"
)

func main() {
    // 構建 URL 和查詢參數
    baseURL := "https://api.github.com/search/repositories"

    params := url.Values{}
    params.Add("q", "golang")
    params.Add("sort", "stars")
    params.Add("order", "desc")

    fullURL := baseURL + "?" + params.Encode()
    fmt.Println("URL:", fullURL)

    // 發送請求
    resp, err := http.Get(fullURL)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Response length:", len(body))

    // 方法 2: 使用 Request
    req, _ := http.NewRequest("GET", "https://api.github.com/search/repositories", nil)

    q := req.URL.Query()
    q.Add("q", "golang")
    q.Add("sort", "stars")
    req.URL.RawQuery = q.Encode()

    fmt.Println("Constructed URL:", req.URL.String())
}
```

**Node.js (axios):**
```javascript
const axios = require('axios');

async function main() {
    try {
        // 使用 params 選項
        const response = await axios.get('https://api.github.com/search/repositories', {
            params: {
                q: 'golang',
                sort: 'stars',
                order: 'desc'
            }
        });

        console.log("URL:", response.config.url);
        console.log("Response length:", JSON.stringify(response.data).length);

        // 方法 2: 手動構建 URL
        const params = new URLSearchParams({
            q: 'golang',
            sort: 'stars',
            order: 'desc'
        });
        const url = `https://api.github.com/search/repositories?${params}`;
        console.log("Constructed URL:", url);
    } catch (error) {
        console.error("Error:", error.message);
    }
}

main();
```

### 示例 5: 超時和重試

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "time"
)

func makeRequestWithTimeout(url string, timeout time.Duration) error {
    // 創建自定義客戶端
    client := &http.Client{
        Timeout: timeout,
    }

    resp, err := client.Get(url)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Response:", string(body[:100]))
    return nil
}

func makeRequestWithRetry(url string, maxRetries int) error {
    client := &http.Client{
        Timeout: 5 * time.Second,
    }

    var lastErr error
    for i := 0; i < maxRetries; i++ {
        resp, err := client.Get(url)
        if err != nil {
            lastErr = err
            fmt.Printf("Attempt %d failed: %v\n", i+1, err)
            time.Sleep(time.Second * time.Duration(i+1))
            continue
        }
        defer resp.Body.Close()

        if resp.StatusCode == http.StatusOK {
            body, _ := io.ReadAll(resp.Body)
            fmt.Println("Success:", string(body[:100]))
            return nil
        }

        lastErr = fmt.Errorf("status code: %d", resp.StatusCode)
        time.Sleep(time.Second * time.Duration(i+1))
    }

    return fmt.Errorf("max retries exceeded: %v", lastErr)
}

func main() {
    // 超時示例
    fmt.Println("=== Timeout Example ===")
    err := makeRequestWithTimeout("https://httpbin.org/delay/2", 3*time.Second)
    if err != nil {
        fmt.Println("Error:", err)
    }

    // 重試示例
    fmt.Println("\n=== Retry Example ===")
    err = makeRequestWithRetry("https://httpbin.org/status/500", 3)
    if err != nil {
        fmt.Println("Final error:", err)
    }
}
```

**Node.js (axios):**
```javascript
const axios = require('axios');

async function makeRequestWithTimeout(url, timeout) {
    try {
        const response = await axios.get(url, {
            timeout: timeout
        });
        console.log("Response:", response.data.toString().substring(0, 100));
    } catch (error) {
        throw error;
    }
}

async function makeRequestWithRetry(url, maxRetries) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            const response = await axios.get(url, {
                timeout: 5000
            });

            if (response.status === 200) {
                console.log("Success:", JSON.stringify(response.data).substring(0, 100));
                return;
            }
        } catch (error) {
            console.log(`Attempt ${i + 1} failed:`, error.message);
            if (i < maxRetries - 1) {
                await new Promise(resolve => setTimeout(resolve, (i + 1) * 1000));
            } else {
                throw new Error(`Max retries exceeded: ${error.message}`);
            }
        }
    }
}

async function main() {
    // 超時示例
    console.log("=== Timeout Example ===");
    try {
        await makeRequestWithTimeout("https://httpbin.org/delay/2", 3000);
    } catch (error) {
        console.log("Error:", error.message);
    }

    // 重試示例
    console.log("\n=== Retry Example ===");
    try {
        await makeRequestWithRetry("https://httpbin.org/status/500", 3);
    } catch (error) {
        console.log("Final error:", error.message);
    }
}

main();
```

### 示例 6: 文件上傳

**Go:**
```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "mime/multipart"
    "net/http"
    "os"
)

func uploadFile(url, filepath string) error {
    // 打開文件
    file, err := os.Open(filepath)
    if err != nil {
        return err
    }
    defer file.Close()

    // 創建 multipart writer
    body := &bytes.Buffer{}
    writer := multipart.NewWriter(body)

    // 添加文件字段
    part, err := writer.CreateFormFile("file", filepath)
    if err != nil {
        return err
    }

    _, err = io.Copy(part, file)
    if err != nil {
        return err
    }

    // 添加其他字段
    _ = writer.WriteField("description", "Uploaded file")

    // 關閉 writer
    err = writer.Close()
    if err != nil {
        return err
    }

    // 創建請求
    req, err := http.NewRequest("POST", url, body)
    if err != nil {
        return err
    }

    req.Header.Set("Content-Type", writer.FormDataContentType())

    // 發送請求
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    respBody, _ := io.ReadAll(resp.Body)
    fmt.Println("Status:", resp.Status)
    fmt.Println("Response:", string(respBody))

    return nil
}

func main() {
    // 創建測試文件
    testFile := "test.txt"
    os.WriteFile(testFile, []byte("Hello, this is a test file!"), 0644)
    defer os.Remove(testFile)

    // 上傳文件
    err := uploadFile("https://httpbin.org/post", testFile)
    if err != nil {
        fmt.Println("Upload error:", err)
    }
}
```

**Node.js (axios):**
```javascript
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

async function uploadFile(url, filepath) {
    try {
        // 創建 FormData
        const form = new FormData();
        form.append('file', fs.createReadStream(filepath));
        form.append('description', 'Uploaded file');

        // 發送請求
        const response = await axios.post(url, form, {
            headers: form.getHeaders()
        });

        console.log("Status:", response.status);
        console.log("Response:", response.data);
    } catch (error) {
        console.error("Upload error:", error.message);
    }
}

async function main() {
    // 創建測試文件
    const testFile = 'test.txt';
    fs.writeFileSync(testFile, 'Hello, this is a test file!');

    // 上傳文件
    await uploadFile('https://httpbin.org/post', testFile);

    // 清理
    fs.unlinkSync(testFile);
}

main();
```

### 示例 7: Cookie 處理

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "net/http/cookiejar"
    "net/url"
)

func main() {
    // 創建 cookie jar
    jar, err := cookiejar.New(nil)
    if err != nil {
        fmt.Println("Error creating cookie jar:", err)
        return
    }

    // 創建帶 cookie jar 的客戶端
    client := &http.Client{
        Jar: jar,
    }

    // 第一個請求（設置 cookie）
    resp1, err := client.Get("https://httpbin.org/cookies/set?session=abc123")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp1.Body.Close()
    io.ReadAll(resp1.Body)

    // 查看 cookies
    u, _ := url.Parse("https://httpbin.org")
    cookies := jar.Cookies(u)
    fmt.Println("Cookies:")
    for _, cookie := range cookies {
        fmt.Printf("  %s = %s\n", cookie.Name, cookie.Value)
    }

    // 第二個請求（自動發送 cookie）
    resp2, err := client.Get("https://httpbin.org/cookies")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp2.Body.Close()

    body, _ := io.ReadAll(resp2.Body)
    fmt.Println("Response:", string(body))
}
```

**Node.js (axios):**
```javascript
const axios = require('axios');
const tough = require('tough-cookie');
const { wrapper } = require('axios-cookiejar-support');

async function main() {
    // 創建 cookie jar
    const cookieJar = new tough.CookieJar();

    // 創建帶 cookie jar 的客戶端
    const client = wrapper(axios.create({
        jar: cookieJar,
        withCredentials: true
    }));

    try {
        // 第一個請求（設置 cookie）
        await client.get('https://httpbin.org/cookies/set?session=abc123');

        // 查看 cookies
        console.log("Cookies:");
        const cookies = await cookieJar.getCookies('https://httpbin.org');
        cookies.forEach(cookie => {
            console.log(`  ${cookie.key} = ${cookie.value}`);
        });

        // 第二個請求（自動發送 cookie）
        const response = await client.get('https://httpbin.org/cookies');
        console.log("Response:", response.data);
    } catch (error) {
        console.error("Error:", error.message);
    }
}

main();
```

### 示例 8: 代理設置

**Go:**
```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "net/url"
)

func main() {
    // 設置代理
    proxyURL, err := url.Parse("http://proxy.example.com:8080")
    if err != nil {
        fmt.Println("Error parsing proxy URL:", err)
        return
    }

    // 創建自定義 Transport
    transport := &http.Transport{
        Proxy: http.ProxyURL(proxyURL),
    }

    // 創建客戶端
    client := &http.Client{
        Transport: transport,
    }

    // 發送請求
    resp, err := client.Get("https://api.ipify.org?format=json")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("Response:", string(body))

    // 從環境變量讀取代理
    transport2 := &http.Transport{
        Proxy: http.ProxyFromEnvironment,
    }

    client2 := &http.Client{
        Transport: transport2,
    }

    resp2, err := client2.Get("https://api.ipify.org?format=json")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    defer resp2.Body.Close()

    body2, _ := io.ReadAll(resp2.Body)
    fmt.Println("Response (env proxy):", string(body2))
}
```

**Node.js (axios):**
```javascript
const axios = require('axios');

async function main() {
    try {
        // 設置代理
        const response1 = await axios.get('https://api.ipify.org?format=json', {
            proxy: {
                host: 'proxy.example.com',
                port: 8080
            }
        });
        console.log("Response:", response1.data);

        // 從環境變量讀取代理（axios 自動支持）
        // 設置環境變量: HTTP_PROXY, HTTPS_PROXY
        const response2 = await axios.get('https://api.ipify.org?format=json');
        console.log("Response (env proxy):", response2.data);
    } catch (error) {
        console.error("Error:", error.message);
    }
}

main();
```

## 重點總結

1. **基本請求**
   - `http.Get()` - 簡單 GET 請求
   - `http.Post()` - POST 請求
   - `http.NewRequest()` + `client.Do()` - 自定義請求

2. **Response 處理**
   - 必須 `defer resp.Body.Close()`
   - 使用 `io.ReadAll()` 讀取 body
   - 檢查 `StatusCode` 處理錯誤

3. **Client 配置**
   - 設置 Timeout
   - 配置 Transport（代理、連接池等）
   - 使用 CookieJar 管理 cookies

4. **與 Node.js 的主要差異**
   - Go 需要手動關閉 Body
   - Go 的錯誤處理更明確
   - Go 需要手動序列化/反序列化 JSON
   - Go 的 Client 配置更底層

5. **最佳實踐**
   - 總是關閉 Response.Body
   - 使用自定義 Client 而不是默認客戶端
   - 設置適當的超時
   - 檢查狀態碼
   - 處理所有錯誤

## 練習題

### 練習 1: API 客戶端
創建一個簡單的 API 客戶端，支持 GET、POST、PUT、DELETE 方法。

### 練習 2: 重試機制
實現帶指數退避的重試機制。

### 練習 3: 並發請求
同時發送多個請求並等待所有請求完成。

### 練習 4: 下載文件
實現文件下載功能，顯示下載進度。

### 練習 5: 速率限制
實現 API 客戶端的速率限制（每秒最多 N 個請求）。

### 練習 6: 緩存
實現 HTTP 響應緩存機制。

### 練習 7: 認證
實現 OAuth 2.0 或 JWT 認證客戶端。

### 練習 8: WebSocket 客戶端
使用 gorilla/websocket 創建 WebSocket 客戶端。

## 參考答案

### 練習 1 答案
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"
)

type APIClient struct {
    BaseURL    string
    HTTPClient *http.Client
    Headers    map[string]string
}

func NewAPIClient(baseURL string) *APIClient {
    return &APIClient{
        BaseURL: baseURL,
        HTTPClient: &http.Client{
            Timeout: 10 * time.Second,
        },
        Headers: make(map[string]string),
    }
}

func (c *APIClient) doRequest(method, path string, body interface{}) ([]byte, error) {
    var bodyReader io.Reader
    if body != nil {
        jsonData, err := json.Marshal(body)
        if err != nil {
            return nil, err
        }
        bodyReader = bytes.NewBuffer(jsonData)
    }

    req, err := http.NewRequest(method, c.BaseURL+path, bodyReader)
    if err != nil {
        return nil, err
    }

    // 設置 headers
    for key, value := range c.Headers {
        req.Header.Set(key, value)
    }
    if body != nil {
        req.Header.Set("Content-Type", "application/json")
    }

    resp, err := c.HTTPClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    if resp.StatusCode >= 400 {
        return nil, fmt.Errorf("HTTP %d: %s", resp.StatusCode, string(respBody))
    }

    return respBody, nil
}

func (c *APIClient) Get(path string) ([]byte, error) {
    return c.doRequest("GET", path, nil)
}

func (c *APIClient) Post(path string, body interface{}) ([]byte, error) {
    return c.doRequest("POST", path, body)
}

func (c *APIClient) Put(path string, body interface{}) ([]byte, error) {
    return c.doRequest("PUT", path, body)
}

func (c *APIClient) Delete(path string) ([]byte, error) {
    return c.doRequest("DELETE", path, nil)
}

func main() {
    client := NewAPIClient("https://jsonplaceholder.typicode.com")

    // GET
    data, _ := client.Get("/users/1")
    fmt.Println("GET:", string(data))

    // POST
    newUser := map[string]interface{}{
        "name":  "Alice",
        "email": "alice@example.com",
    }
    data, _ = client.Post("/users", newUser)
    fmt.Println("POST:", string(data))
}
```
