# Chapter 54: WebSocket 實時通信

## 概述

WebSocket 提供了客戶端和服務器之間的全雙工、雙向通信通道。本章將介紹如何在 Go 中使用 gorilla/websocket 實現 WebSocket 服務器，並與 Node.js 的 Socket.io 進行對比。

## WebSocket vs HTTP 輪詢

### HTTP 長輪詢
```
客戶端 → 請求 → 服務器
      ← 等待 ←
      ← 響應 ←
客戶端 → 新請求 → 服務器
```

**缺點**：
- 頻繁的連接建立和關閉
- HTTP 頭部開銷大
- 延遲高

### WebSocket
```
客戶端 ←─────────→ 服務器
     持久連接，雙向通信
```

**優點**：
- 低延遲
- 雙向實時通信
- 更少的開銷

## Go WebSocket vs Socket.io 對比

### Socket.io (Node.js)
```javascript
const io = require('socket.io')(server);

io.on('connection', (socket) => {
    socket.on('message', (data) => {
        socket.emit('response', data);
    });
});
```

**特點**：
- 自動降級（WebSocket → 長輪詢）
- 房間和命名空間
- 自動重連
- 廣播支持

### gorilla/websocket (Go)
```go
upgrader := websocket.Upgrader{}
conn, _ := upgrader.Upgrade(w, r, nil)

for {
    _, message, _ := conn.ReadMessage()
    conn.WriteMessage(websocket.TextMessage, message)
}
```

**特點**：
- 輕量級
- 性能高
- 需要手動實現高級功能
- 完全控制

## 完整的 WebSocket 聊天應用

### 項目結構

```
websocket-chat/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── hub/
│   │   └── hub.go
│   ├── client/
│   │   └── client.go
│   └── handlers/
│       └── websocket.go
├── web/
│   ├── index.html
│   └── app.js
└── go.mod
```

### 1. Hub（消息中心）

Hub 負責管理所有活動連接和廣播消息。

#### internal/hub/hub.go
```go
package hub

import (
    "log"
    "sync"
)

// Client 接口
type Client interface {
    GetID() string
    Send(message []byte) error
    Close()
}

// Hub 管理所有客戶端連接
type Hub struct {
    // 已註冊的客戶端
    clients map[string]Client

    // 廣播消息通道
    broadcast chan []byte

    // 註冊新客戶端
    register chan Client

    // 註銷客戶端
    unregister chan Client

    // 保護 clients map 的互斥鎖
    mu sync.RWMutex
}

// NewHub 創建新的 Hub
func NewHub() *Hub {
    return &Hub{
        clients:    make(map[string]Client),
        broadcast:  make(chan []byte, 256),
        register:   make(chan Client),
        unregister: make(chan Client),
    }
}

// Run 啟動 Hub
func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.registerClient(client)

        case client := <-h.unregister:
            h.unregisterClient(client)

        case message := <-h.broadcast:
            h.broadcastMessage(message)
        }
    }
}

// registerClient 註冊客戶端
func (h *Hub) registerClient(client Client) {
    h.mu.Lock()
    defer h.mu.Unlock()

    h.clients[client.GetID()] = client
    log.Printf("Client registered: %s, total: %d", client.GetID(), len(h.clients))
}

// unregisterClient 註銷客戶端
func (h *Hub) unregisterClient(client Client) {
    h.mu.Lock()
    defer h.mu.Unlock()

    if _, ok := h.clients[client.GetID()]; ok {
        delete(h.clients, client.GetID())
        client.Close()
        log.Printf("Client unregistered: %s, total: %d", client.GetID(), len(h.clients))
    }
}

// broadcastMessage 廣播消息給所有客戶端
func (h *Hub) broadcastMessage(message []byte) {
    h.mu.RLock()
    defer h.mu.RUnlock()

    for id, client := range h.clients {
        if err := client.Send(message); err != nil {
            log.Printf("Error sending to client %s: %v", id, err)
        }
    }
}

// Broadcast 發送廣播消息
func (h *Hub) Broadcast(message []byte) {
    h.broadcast <- message
}

// Register 註冊客戶端
func (h *Hub) Register(client Client) {
    h.register <- client
}

// Unregister 註銷客戶端
func (h *Hub) Unregister(client Client) {
    h.unregister <- client
}

// GetClientCount 獲取在線客戶端數量
func (h *Hub) GetClientCount() int {
    h.mu.RLock()
    defer h.mu.RUnlock()
    return len(h.clients)
}

// SendToClient 發送消息給特定客戶端
func (h *Hub) SendToClient(clientID string, message []byte) error {
    h.mu.RLock()
    defer h.mu.RUnlock()

    client, ok := h.clients[clientID]
    if !ok {
        return ErrClientNotFound
    }

    return client.Send(message)
}

var ErrClientNotFound = fmt.Errorf("client not found")
```

### 2. Client（客戶端連接）

#### internal/client/client.go
```go
package client

import (
    "encoding/json"
    "log"
    "time"

    "github.com/google/uuid"
    "github.com/gorilla/websocket"
)

const (
    // 寫入等待時間
    writeWait = 10 * time.Second

    // Pong 等待時間
    pongWait = 60 * time.Second

    // Ping 間隔（必須小於 pongWait）
    pingPeriod = (pongWait * 9) / 10

    // 最大消息大小
    maxMessageSize = 512
)

// Message 消息結構
type Message struct {
    Type      string    `json:"type"`      // message, join, leave, typing
    UserID    string    `json:"user_id"`
    Username  string    `json:"username"`
    Content   string    `json:"content"`
    Timestamp time.Time `json:"timestamp"`
}

// Client WebSocket 客戶端
type Client struct {
    id       string
    hub      Hub
    conn     *websocket.Conn
    send     chan []byte
    username string
}

// Hub 接口
type Hub interface {
    Register(client *Client)
    Unregister(client *Client)
    Broadcast(message []byte)
}

// NewClient 創建新客戶端
func NewClient(hub Hub, conn *websocket.Conn, username string) *Client {
    return &Client{
        id:       uuid.New().String(),
        hub:      hub,
        conn:     conn,
        send:     make(chan []byte, 256),
        username: username,
    }
}

// GetID 獲取客戶端 ID
func (c *Client) GetID() string {
    return c.id
}

// Send 發送消息
func (c *Client) Send(message []byte) error {
    select {
    case c.send <- message:
        return nil
    default:
        return fmt.Errorf("send buffer full")
    }
}

// Close 關閉連接
func (c *Client) Close() {
    close(c.send)
}

// ReadPump 從 WebSocket 連接讀取消息
func (c *Client) ReadPump() {
    defer func() {
        c.hub.Unregister(c)
        c.conn.Close()
    }()

    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })
    c.conn.SetReadLimit(maxMessageSize)

    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, websocket.CloseGoingAway, websocket.CloseAbnormalClosure) {
                log.Printf("WebSocket error: %v", err)
            }
            break
        }

        // 解析消息
        var msg Message
        if err := json.Unmarshal(message, &msg); err != nil {
            log.Printf("Failed to parse message: %v", err)
            continue
        }

        // 添加元數據
        msg.UserID = c.id
        msg.Username = c.username
        msg.Timestamp = time.Now()

        // 重新序列化
        data, err := json.Marshal(msg)
        if err != nil {
            log.Printf("Failed to marshal message: %v", err)
            continue
        }

        // 廣播消息
        c.hub.Broadcast(data)
    }
}

// WritePump 向 WebSocket 連接寫入消息
func (c *Client) WritePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                // Hub 關閉了通道
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            w, err := c.conn.NextWriter(websocket.TextMessage)
            if err != nil {
                return
            }
            w.Write(message)

            // 將隊列中的所有待發送消息一起發送
            n := len(c.send)
            for i := 0; i < n; i++ {
                w.Write([]byte{'\n'})
                w.Write(<-c.send)
            }

            if err := w.Close(); err != nil {
                return
            }

        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

### 3. WebSocket Handler

#### internal/handlers/websocket.go
```go
package handlers

import (
    "log"
    "net/http"

    "websocket-chat/internal/client"
    "websocket-chat/internal/hub"

    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        // 生產環境應該檢查來源
        return true
    },
}

// WebSocketHandler WebSocket 處理器
type WebSocketHandler struct {
    hub *hub.Hub
}

// NewWebSocketHandler 創建 WebSocket 處理器
func NewWebSocketHandler(hub *hub.Hub) *WebSocketHandler {
    return &WebSocketHandler{hub: hub}
}

// ServeWS 處理 WebSocket 連接
func (h *WebSocketHandler) ServeWS(w http.ResponseWriter, r *http.Request) {
    // 升級 HTTP 連接為 WebSocket
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("WebSocket upgrade error: %v", err)
        return
    }

    // 從查詢參數獲取用戶名
    username := r.URL.Query().Get("username")
    if username == "" {
        username = "Anonymous"
    }

    // 創建客戶端
    client := client.NewClient(h.hub, conn, username)

    // 註冊客戶端
    h.hub.Register(client)

    // 啟動讀寫 goroutine
    go client.WritePump()
    go client.ReadPump()

    log.Printf("New WebSocket connection: %s (%s)", client.GetID(), username)
}

// GetStats 獲取統計信息
func (h *WebSocketHandler) GetStats(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]interface{}{
        "online_users": h.hub.GetClientCount(),
    })
}
```

### 4. 主程序

#### cmd/server/main.go
```go
package main

import (
    "log"
    "net/http"

    "websocket-chat/internal/handlers"
    "websocket-chat/internal/hub"

    "github.com/gorilla/mux"
)

func main() {
    // 創建 Hub
    chatHub := hub.NewHub()
    go chatHub.Run()

    // 創建處理器
    wsHandler := handlers.NewWebSocketHandler(chatHub)

    // 設置路由
    r := mux.NewRouter()

    // WebSocket 端點
    r.HandleFunc("/ws", wsHandler.ServeWS)

    // API 端點
    r.HandleFunc("/api/stats", wsHandler.GetStats).Methods("GET")

    // 靜態文件
    r.PathPrefix("/").Handler(http.FileServer(http.Dir("./web")))

    // 啟動服務器
    addr := ":8080"
    log.Printf("WebSocket server starting on %s", addr)
    if err := http.ListenAndServe(addr, r); err != nil {
        log.Fatal(err)
    }
}
```

### 5. 前端代碼

#### web/index.html
```html
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WebSocket Chat</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: Arial, sans-serif; }

        .container {
            max-width: 800px;
            margin: 50px auto;
            background: #f5f5f5;
            border-radius: 10px;
            overflow: hidden;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
        }

        .header {
            background: #4CAF50;
            color: white;
            padding: 20px;
            text-align: center;
        }

        #messages {
            height: 400px;
            overflow-y: auto;
            padding: 20px;
            background: white;
        }

        .message {
            margin-bottom: 10px;
            padding: 10px;
            border-radius: 5px;
            background: #e3f2fd;
        }

        .message.own {
            background: #c8e6c9;
            text-align: right;
        }

        .message .username {
            font-weight: bold;
            color: #1976D2;
        }

        .message .time {
            font-size: 0.8em;
            color: #666;
        }

        .input-area {
            padding: 20px;
            background: #fafafa;
            display: flex;
            gap: 10px;
        }

        input {
            flex: 1;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }

        button {
            padding: 10px 20px;
            background: #4CAF50;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }

        button:hover {
            background: #45a049;
        }

        .status {
            padding: 10px;
            text-align: center;
            background: #fff3cd;
            color: #856404;
        }

        .status.connected {
            background: #d4edda;
            color: #155724;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>WebSocket Chat Room</h1>
            <div id="status" class="status">Connecting...</div>
        </div>

        <div id="messages"></div>

        <div class="input-area">
            <input type="text" id="messageInput" placeholder="Type a message..." />
            <button onclick="sendMessage()">Send</button>
        </div>
    </div>

    <script src="app.js"></script>
</body>
</html>
```

#### web/app.js
```javascript
let ws;
let username = prompt("Enter your username:") || "Anonymous";

function connect() {
    ws = new WebSocket(`ws://${window.location.host}/ws?username=${encodeURIComponent(username)}`);

    ws.onopen = function() {
        console.log("Connected to WebSocket");
        updateStatus("Connected", true);

        // 發送加入消息
        sendSystemMessage("join", `${username} joined the chat`);
    };

    ws.onmessage = function(event) {
        const message = JSON.parse(event.data);
        displayMessage(message);
    };

    ws.onclose = function() {
        console.log("WebSocket connection closed");
        updateStatus("Disconnected", false);

        // 5 秒後重連
        setTimeout(connect, 5000);
    };

    ws.onerror = function(error) {
        console.error("WebSocket error:", error);
    };
}

function sendMessage() {
    const input = document.getElementById('messageInput');
    const content = input.value.trim();

    if (content && ws.readyState === WebSocket.OPEN) {
        const message = {
            type: 'message',
            content: content
        };

        ws.send(JSON.stringify(message));
        input.value = '';
    }
}

function sendSystemMessage(type, content) {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type, content }));
    }
}

function displayMessage(message) {
    const messagesDiv = document.getElementById('messages');
    const messageDiv = document.createElement('div');
    messageDiv.className = 'message';

    if (message.username === username) {
        messageDiv.classList.add('own');
    }

    const time = new Date(message.timestamp).toLocaleTimeString();

    messageDiv.innerHTML = `
        <div class="username">${message.username}</div>
        <div class="content">${message.content}</div>
        <div class="time">${time}</div>
    `;

    messagesDiv.appendChild(messageDiv);
    messagesDiv.scrollTop = messagesDiv.scrollHeight;
}

function updateStatus(text, connected) {
    const status = document.getElementById('status');
    status.textContent = text;
    status.className = 'status' + (connected ? ' connected' : '');
}

// Enter 鍵發送消息
document.getElementById('messageInput').addEventListener('keypress', function(e) {
    if (e.key === 'Enter') {
        sendMessage();
    }
});

// 啟動連接
connect();
```

## 高級功能

### 1. 房間功能

```go
package hub

type Room struct {
    id      string
    clients map[string]*Client
    hub     *Hub
}

type Hub struct {
    rooms map[string]*Room
    // ... 其他字段
}

func (h *Hub) JoinRoom(client *Client, roomID string) {
    room, ok := h.rooms[roomID]
    if !ok {
        room = &Room{
            id:      roomID,
            clients: make(map[string]*Client),
            hub:     h,
        }
        h.rooms[roomID] = room
    }

    room.clients[client.id] = client
}

func (r *Room) Broadcast(message []byte) {
    for _, client := range r.clients {
        client.Send(message)
    }
}
```

### 2. 私聊功能

```go
type DirectMessage struct {
    From    string `json:"from"`
    To      string `json:"to"`
    Content string `json:"content"`
}

func (h *Hub) SendDirectMessage(msg *DirectMessage) error {
    h.mu.RLock()
    defer h.mu.RUnlock()

    toClient, ok := h.clients[msg.To]
    if !ok {
        return fmt.Errorf("recipient not found")
    }

    data, _ := json.Marshal(msg)
    return toClient.Send(data)
}
```

### 3. 在線用戶列表

```go
func (h *Hub) GetOnlineUsers() []UserInfo {
    h.mu.RLock()
    defer h.mu.RUnlock()

    users := make([]UserInfo, 0, len(h.clients))
    for _, client := range h.clients {
        users = append(users, UserInfo{
            ID:       client.GetID(),
            Username: client.username,
        })
    }

    return users
}
```

## Node.js Socket.io 對應版本

### 服務器

```javascript
const express = require('express');
const http = require('http');
const socketIO = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketIO(server);

// 在線用戶
const users = new Map();

io.on('connection', (socket) => {
    console.log('New connection:', socket.id);

    // 用戶加入
    socket.on('join', (username) => {
        users.set(socket.id, username);

        // 廣播用戶加入
        io.emit('user-joined', {
            userId: socket.id,
            username: username,
            timestamp: new Date()
        });

        // 發送在線用戶列表
        socket.emit('online-users', Array.from(users.entries()));
    });

    // 接收消息
    socket.on('message', (data) => {
        const message = {
            userId: socket.id,
            username: users.get(socket.id),
            content: data.content,
            timestamp: new Date()
        };

        // 廣播消息
        io.emit('message', message);
    });

    // 私聊
    socket.on('direct-message', ({ to, content }) => {
        const message = {
            from: socket.id,
            username: users.get(socket.id),
            content: content,
            timestamp: new Date()
        };

        // 發送給特定用戶
        io.to(to).emit('direct-message', message);
    });

    // 加入房間
    socket.on('join-room', (room) => {
        socket.join(room);
        io.to(room).emit('user-joined-room', {
            userId: socket.id,
            username: users.get(socket.id),
            room: room
        });
    });

    // 房間消息
    socket.on('room-message', ({ room, content }) => {
        io.to(room).emit('room-message', {
            userId: socket.id,
            username: users.get(socket.id),
            content: content,
            timestamp: new Date()
        });
    });

    // 用戶斷開
    socket.on('disconnect', () => {
        const username = users.get(socket.id);
        users.delete(socket.id);

        io.emit('user-left', {
            userId: socket.id,
            username: username,
            timestamp: new Date()
        });

        console.log('User disconnected:', socket.id);
    });
});

app.use(express.static('public'));

server.listen(3000, () => {
    console.log('Server running on http://localhost:3000');
});
```

### 客戶端

```javascript
const socket = io();

socket.on('connect', () => {
    console.log('Connected to server');
    socket.emit('join', username);
});

socket.on('message', (message) => {
    displayMessage(message);
});

socket.on('user-joined', (data) => {
    displaySystemMessage(`${data.username} joined`);
});

socket.on('user-left', (data) => {
    displaySystemMessage(`${data.username} left`);
});

function sendMessage(content) {
    socket.emit('message', { content });
}
```

## 重點總結

### WebSocket 核心概念

1. **連接升級**
   - HTTP → WebSocket
   - 握手協議

2. **雙向通信**
   - 客戶端和服務器都可以主動發送消息
   - 全雙工通信

3. **連接管理**
   - Ping/Pong 保活
   - 自動重連
   - 優雅關閉

### Go vs Node.js WebSocket 對比

| 特性 | gorilla/websocket | Socket.io |
|------|-------------------|-----------|
| 性能 | 高 | 中 |
| 內存使用 | 低 | 高 |
| 並發處理 | Goroutine（優秀）| 事件循環 |
| 自動降級 | 無 | 有 |
| 房間支持 | 需自己實現 | 內置 |
| 學習曲線 | 中等 | 簡單 |

### 最佳實踐

1. **安全性**
   - 驗證 Origin
   - 認證和授權
   - 限制消息大小
   - 速率限制

2. **性能優化**
   - 使用消息池
   - 批量發送消息
   - 壓縮大消息

3. **錯誤處理**
   - 優雅的連接關閉
   - 自動重連
   - 錯誤日誌

4. **監控**
   - 在線用戶數
   - 消息吞吐量
   - 連接時長

## 練習題

### 練習 1: 實現房間功能

添加聊天室功能，用戶可以加入不同的房間。

### 練習 2: 添加認證

為 WebSocket 連接添加 JWT 認證。

```go
func (h *WebSocketHandler) ServeWS(w http.ResponseWriter, r *http.Request) {
    // 驗證 token
    token := r.URL.Query().Get("token")
    // 驗證邏輯...
}
```

### 練習 3: 消息持久化

將聊天消息保存到數據庫，並支持歷史消息加載。

### 練習 4: 在線狀態

實現用戶在線狀態（在線、離開、忙碌）。

### 練習 5: 文件傳輸

實現通過 WebSocket 傳輸文件功能。

---

下一章：[Chapter 55: JWT 認證](55-jwt-auth.md)
