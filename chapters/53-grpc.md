# Chapter 53: gRPC 與 Protocol Buffers

## 概述

gRPC 是 Google 開發的高性能、開源的 RPC（遠程過程調用）框架。它使用 Protocol Buffers 作為接口定義語言，提供了比 REST 更高效的服務間通信方式。本章將介紹如何在 Go 中使用 gRPC 構建微服務。

## gRPC vs REST 對比

### REST API
```
優點：
- 易於理解和使用
- 廣泛的工具支持
- 瀏覽器友好
- 人類可讀（JSON）

缺點：
- 性能較低
- 序列化開銷大
- 沒有強類型約束
```

### gRPC
```
優點：
- 高性能（基於 HTTP/2）
- 強類型約束
- 雙向流支持
- 多語言支持

缺點：
- 學習曲線較陡
- 調試較困難
- 瀏覽器支持有限
```

## Protocol Buffers 基礎

Protocol Buffers（protobuf）是一種語言中立、平台中立的數據序列化格式。

### 基本語法

#### user.proto
```protobuf
syntax = "proto3";

package user;

option go_package = "github.com/yourname/grpc-demo/proto/user";

// 用戶消息
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}

// 創建用戶請求
message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
}

// 創建用戶響應
message CreateUserResponse {
  User user = 1;
  string message = 2;
}

// 獲取用戶請求
message GetUserRequest {
  int32 id = 1;
}

// 列表用戶請求
message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}

// 列表用戶響應
message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
}

// 用戶服務定義
service UserService {
  // 創建用戶
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);

  // 獲取用戶
  rpc GetUser(GetUserRequest) returns (User);

  // 列出用戶
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);

  // 刪除用戶
  rpc DeleteUser(GetUserRequest) returns (DeleteUserResponse);

  // 流式：實時用戶更新（服務器流）
  rpc StreamUsers(ListUsersRequest) returns (stream User);

  // 流式：批量創建用戶（客戶端流）
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateUsersResponse);

  // 流式：雙向聊天
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message DeleteUserResponse {
  bool success = 1;
  string message = 2;
}

message BatchCreateUsersResponse {
  int32 count = 1;
  repeated User users = 2;
}

message ChatMessage {
  int32 user_id = 1;
  string message = 2;
  int64 timestamp = 3;
}
```

### 數據類型對照

| Protocol Buffers | Go | Node.js |
|------------------|-----|---------|
| double | float64 | number |
| float | float32 | number |
| int32 | int32 | number |
| int64 | int64 | number/string |
| bool | bool | boolean |
| string | string | string |
| bytes | []byte | Buffer |
| repeated | slice | Array |
| map | map | Object/Map |

## 完整的 gRPC 項目

### 項目結構

```
grpc-user-service/
├── proto/
│   └── user/
│       └── user.proto
├── cmd/
│   ├── server/
│   │   └── main.go
│   └── client/
│       └── main.go
├── internal/
│   ├── server/
│   │   └── user_server.go
│   └── repository/
│       └── user_repository.go
├── Makefile
└── go.mod
```

### 1. 編譯 Protocol Buffers

#### Makefile
```makefile
.PHONY: proto clean

# 生成 protobuf 代碼
proto:
	protoc --go_out=. --go_opt=paths=source_relative \
		--go-grpc_out=. --go-grpc_opt=paths=source_relative \
		proto/user/user.proto

# 安裝 protoc 插件
install-tools:
	go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
	go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# 運行服務器
run-server:
	go run cmd/server/main.go

# 運行客戶端
run-client:
	go run cmd/client/main.go

clean:
	rm -f proto/user/*.pb.go
```

### 2. 實現 gRPC 服務器

#### internal/server/user_server.go
```go
package server

import (
    "context"
    "fmt"
    "io"
    "log"
    "time"

    pb "grpc-user-service/proto/user"
    "grpc-user-service/internal/repository"

    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type UserServer struct {
    pb.UnimplementedUserServiceServer
    repo *repository.UserRepository
}

func NewUserServer(repo *repository.UserRepository) *UserServer {
    return &UserServer{repo: repo}
}

// CreateUser 創建用戶
func (s *UserServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.CreateUserResponse, error) {
    log.Printf("CreateUser called: name=%s, email=%s", req.Name, req.Email)

    // 驗證輸入
    if req.Name == "" || req.Email == "" {
        return nil, status.Error(codes.InvalidArgument, "name and email are required")
    }

    // 創建用戶
    user, err := s.repo.Create(req.Name, req.Email, req.Password)
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to create user: %v", err)
    }

    return &pb.CreateUserResponse{
        User: &pb.User{
            Id:        user.ID,
            Name:      user.Name,
            Email:     user.Email,
            CreatedAt: user.CreatedAt.Unix(),
        },
        Message: "User created successfully",
    }, nil
}

// GetUser 獲取用戶
func (s *UserServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    log.Printf("GetUser called: id=%d", req.Id)

    user, err := s.repo.GetByID(int(req.Id))
    if err != nil {
        return nil, status.Error(codes.NotFound, "user not found")
    }

    return &pb.User{
        Id:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt.Unix(),
    }, nil
}

// ListUsers 列出用戶
func (s *UserServer) ListUsers(ctx context.Context, req *pb.ListUsersRequest) (*pb.ListUsersResponse, error) {
    log.Printf("ListUsers called: page=%d, page_size=%d", req.Page, req.PageSize)

    users, total, err := s.repo.List(int(req.Page), int(req.PageSize))
    if err != nil {
        return nil, status.Errorf(codes.Internal, "failed to list users: %v", err)
    }

    pbUsers := make([]*pb.User, len(users))
    for i, user := range users {
        pbUsers[i] = &pb.User{
            Id:        user.ID,
            Name:      user.Name,
            Email:     user.Email,
            CreatedAt: user.CreatedAt.Unix(),
        }
    }

    return &pb.ListUsersResponse{
        Users: pbUsers,
        Total: int32(total),
    }, nil
}

// DeleteUser 刪除用戶
func (s *UserServer) DeleteUser(ctx context.Context, req *pb.GetUserRequest) (*pb.DeleteUserResponse, error) {
    log.Printf("DeleteUser called: id=%d", req.Id)

    err := s.repo.Delete(int(req.Id))
    if err != nil {
        return nil, status.Error(codes.NotFound, "user not found")
    }

    return &pb.DeleteUserResponse{
        Success: true,
        Message: "User deleted successfully",
    }, nil
}

// StreamUsers 服務器流式傳輸用戶（演示服務器流）
func (s *UserServer) StreamUsers(req *pb.ListUsersRequest, stream pb.UserService_StreamUsersServer) error {
    log.Printf("StreamUsers called: page=%d, page_size=%d", req.Page, req.PageSize)

    users, _, err := s.repo.List(int(req.Page), int(req.PageSize))
    if err != nil {
        return status.Errorf(codes.Internal, "failed to list users: %v", err)
    }

    // 逐個發送用戶，模擬流式傳輸
    for _, user := range users {
        pbUser := &pb.User{
            Id:        user.ID,
            Name:      user.Name,
            Email:     user.Email,
            CreatedAt: user.CreatedAt.Unix(),
        }

        if err := stream.Send(pbUser); err != nil {
            return status.Errorf(codes.Internal, "failed to send user: %v", err)
        }

        // 模擬延遲
        time.Sleep(100 * time.Millisecond)
    }

    return nil
}

// BatchCreateUsers 批量創建用戶（演示客戶端流）
func (s *UserServer) BatchCreateUsers(stream pb.UserService_BatchCreateUsersServer) error {
    log.Println("BatchCreateUsers called")

    var users []*pb.User
    count := 0

    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // 客戶端發送完畢
            return stream.SendAndClose(&pb.BatchCreateUsersResponse{
                Count: int32(count),
                Users: users,
            })
        }
        if err != nil {
            return status.Errorf(codes.Internal, "failed to receive: %v", err)
        }

        // 創建用戶
        user, err := s.repo.Create(req.Name, req.Email, req.Password)
        if err != nil {
            log.Printf("Failed to create user: %v", err)
            continue
        }

        pbUser := &pb.User{
            Id:        user.ID,
            Name:      user.Name,
            Email:     user.Email,
            CreatedAt: user.CreatedAt.Unix(),
        }

        users = append(users, pbUser)
        count++
    }
}

// Chat 雙向流式聊天（演示雙向流）
func (s *UserServer) Chat(stream pb.UserService_ChatServer) error {
    log.Println("Chat stream started")

    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return status.Errorf(codes.Internal, "failed to receive: %v", err)
        }

        log.Printf("Received message from user %d: %s", msg.UserId, msg.Message)

        // 回覆消息
        response := &pb.ChatMessage{
            UserId:    0, // 服務器 ID
            Message:   fmt.Sprintf("Echo: %s", msg.Message),
            Timestamp: time.Now().Unix(),
        }

        if err := stream.Send(response); err != nil {
            return status.Errorf(codes.Internal, "failed to send: %v", err)
        }
    }
}
```

#### cmd/server/main.go
```go
package main

import (
    "database/sql"
    "log"
    "net"

    pb "grpc-user-service/proto/user"
    "grpc-user-service/internal/repository"
    "grpc-user-service/internal/server"

    _ "github.com/lib/pq"
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
)

func main() {
    // 連接數據庫
    db, err := sql.Open("postgres", "postgres://postgres:postgres@localhost/userdb?sslmode=disable")
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }
    defer db.Close()

    // 初始化 repository
    repo := repository.NewUserRepository(db)

    // 創建 gRPC 服務器
    grpcServer := grpc.NewServer()

    // 註冊用戶服務
    userServer := server.NewUserServer(repo)
    pb.RegisterUserServiceServer(grpcServer, userServer)

    // 註冊反射服務（用於 grpcurl 等工具）
    reflection.Register(grpcServer)

    // 監聽端口
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("Failed to listen: %v", err)
    }

    log.Println("gRPC server starting on :50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("Failed to serve: %v", err)
    }
}
```

### 3. 實現 gRPC 客戶端

#### cmd/client/main.go
```go
package main

import (
    "context"
    "fmt"
    "io"
    "log"
    "time"

    pb "grpc-user-service/proto/user"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    // 連接到 gRPC 服務器
    conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()

    client := pb.NewUserServiceClient(conn)

    // 測試各種 RPC 方法
    testUnaryRPC(client)
    testServerStreamingRPC(client)
    testClientStreamingRPC(client)
    testBidirectionalStreamingRPC(client)
}

// 測試一元 RPC
func testUnaryRPC(client pb.UserServiceClient) {
    fmt.Println("\n=== Testing Unary RPC ===")

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // 1. 創建用戶
    createResp, err := client.CreateUser(ctx, &pb.CreateUserRequest{
        Name:     "Alice",
        Email:    "alice@example.com",
        Password: "secret123",
    })
    if err != nil {
        log.Printf("CreateUser failed: %v", err)
        return
    }
    fmt.Printf("Created user: %+v\n", createResp.User)

    // 2. 獲取用戶
    user, err := client.GetUser(ctx, &pb.GetUserRequest{
        Id: createResp.User.Id,
    })
    if err != nil {
        log.Printf("GetUser failed: %v", err)
        return
    }
    fmt.Printf("Got user: %+v\n", user)

    // 3. 列出用戶
    listResp, err := client.ListUsers(ctx, &pb.ListUsersRequest{
        Page:     1,
        PageSize: 10,
    })
    if err != nil {
        log.Printf("ListUsers failed: %v", err)
        return
    }
    fmt.Printf("Total users: %d\n", listResp.Total)
    for _, u := range listResp.Users {
        fmt.Printf("  - %s (%s)\n", u.Name, u.Email)
    }
}

// 測試服務器流式 RPC
func testServerStreamingRPC(client pb.UserServiceClient) {
    fmt.Println("\n=== Testing Server Streaming RPC ===")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    stream, err := client.StreamUsers(ctx, &pb.ListUsersRequest{
        Page:     1,
        PageSize: 5,
    })
    if err != nil {
        log.Printf("StreamUsers failed: %v", err)
        return
    }

    for {
        user, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Printf("Receive error: %v", err)
            break
        }
        fmt.Printf("Streamed user: %s (%s)\n", user.Name, user.Email)
    }
}

// 測試客戶端流式 RPC
func testClientStreamingRPC(client pb.UserServiceClient) {
    fmt.Println("\n=== Testing Client Streaming RPC ===")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    stream, err := client.BatchCreateUsers(ctx)
    if err != nil {
        log.Printf("BatchCreateUsers failed: %v", err)
        return
    }

    // 發送多個創建請求
    users := []*pb.CreateUserRequest{
        {Name: "Bob", Email: "bob@example.com", Password: "pass123"},
        {Name: "Charlie", Email: "charlie@example.com", Password: "pass123"},
        {Name: "David", Email: "david@example.com", Password: "pass123"},
    }

    for _, u := range users {
        if err := stream.Send(u); err != nil {
            log.Printf("Send error: %v", err)
            return
        }
        fmt.Printf("Sent create request for: %s\n", u.Name)
        time.Sleep(100 * time.Millisecond)
    }

    resp, err := stream.CloseAndRecv()
    if err != nil {
        log.Printf("CloseAndRecv error: %v", err)
        return
    }

    fmt.Printf("Batch created %d users\n", resp.Count)
}

// 測試雙向流式 RPC
func testBidirectionalStreamingRPC(client pb.UserServiceClient) {
    fmt.Println("\n=== Testing Bidirectional Streaming RPC ===")

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    stream, err := client.Chat(ctx)
    if err != nil {
        log.Printf("Chat failed: %v", err)
        return
    }

    // 啟動接收 goroutine
    waitc := make(chan struct{})
    go func() {
        for {
            msg, err := stream.Recv()
            if err == io.EOF {
                close(waitc)
                return
            }
            if err != nil {
                log.Printf("Receive error: %v", err)
                close(waitc)
                return
            }
            fmt.Printf("Received: %s\n", msg.Message)
        }
    }()

    // 發送消息
    messages := []string{"Hello", "How are you?", "Goodbye"}
    for _, msg := range messages {
        if err := stream.Send(&pb.ChatMessage{
            UserId:    1,
            Message:   msg,
            Timestamp: time.Now().Unix(),
        }); err != nil {
            log.Printf("Send error: %v", err)
            return
        }
        fmt.Printf("Sent: %s\n", msg)
        time.Sleep(500 * time.Millisecond)
    }

    stream.CloseSend()
    <-waitc
}
```

## gRPC 中間件

### 日誌攔截器

```go
package interceptor

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
)

// UnaryServerInterceptor 一元 RPC 攔截器
func UnaryServerInterceptor() grpc.UnaryServerInterceptor {
    return func(
        ctx context.Context,
        req interface{},
        info *grpc.UnaryServerInfo,
        handler grpc.UnaryHandler,
    ) (interface{}, error) {
        start := time.Now()

        // 調用實際的 handler
        resp, err := handler(ctx, req)

        // 記錄日誌
        log.Printf(
            "method=%s duration=%s error=%v",
            info.FullMethod,
            time.Since(start),
            err,
        )

        return resp, err
    }
}

// StreamServerInterceptor 流式 RPC 攔截器
func StreamServerInterceptor() grpc.StreamServerInterceptor {
    return func(
        srv interface{},
        ss grpc.ServerStream,
        info *grpc.StreamServerInfo,
        handler grpc.StreamHandler,
    ) error {
        start := time.Now()

        err := handler(srv, ss)

        log.Printf(
            "method=%s stream=%v duration=%s error=%v",
            info.FullMethod,
            info.IsServerStream,
            time.Since(start),
            err,
        )

        return err
    }
}
```

### 使用攔截器

```go
func main() {
    // 創建帶攔截器的 gRPC 服務器
    grpcServer := grpc.NewServer(
        grpc.UnaryInterceptor(interceptor.UnaryServerInterceptor()),
        grpc.StreamInterceptor(interceptor.StreamServerInterceptor()),
    )

    // ... 註冊服務
}
```

## Node.js 中使用 gRPC

### 安裝依賴

```bash
npm install @grpc/grpc-js @grpc/proto-loader
```

### 服務器實現

```javascript
// server.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const PROTO_PATH = './proto/user/user.proto';

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

// 實現服務方法
function createUser(call, callback) {
    const { name, email, password } = call.request;

    // 創建用戶邏輯...
    const user = {
        id: 1,
        name,
        email,
        created_at: Date.now() / 1000
    };

    callback(null, {
        user,
        message: 'User created successfully'
    });
}

function getUser(call, callback) {
    const { id } = call.request;

    // 獲取用戶邏輯...
    const user = {
        id,
        name: 'Alice',
        email: 'alice@example.com',
        created_at: Date.now() / 1000
    };

    callback(null, user);
}

function listUsers(call, callback) {
    const { page, page_size } = call.request;

    // 獲取用戶列表...
    const users = [
        { id: 1, name: 'Alice', email: 'alice@example.com', created_at: Date.now() / 1000 }
    ];

    callback(null, { users, total: users.length });
}

// 創建服務器
function main() {
    const server = new grpc.Server();

    server.addService(userProto.UserService.service, {
        createUser,
        getUser,
        listUsers
    });

    server.bindAsync(
        '0.0.0.0:50051',
        grpc.ServerCredentials.createInsecure(),
        (err, port) => {
            if (err) {
                console.error(err);
                return;
            }
            console.log(`Server running on port ${port}`);
            server.start();
        }
    );
}

main();
```

### 客戶端實現

```javascript
// client.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const PROTO_PATH = './proto/user/user.proto';

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

function main() {
    const client = new userProto.UserService(
        'localhost:50051',
        grpc.credentials.createInsecure()
    );

    // 創建用戶
    client.createUser({
        name: 'Alice',
        email: 'alice@example.com',
        password: 'secret123'
    }, (err, response) => {
        if (err) {
            console.error(err);
            return;
        }
        console.log('User created:', response.user);
    });

    // 獲取用戶
    client.getUser({ id: 1 }, (err, user) => {
        if (err) {
            console.error(err);
            return;
        }
        console.log('Got user:', user);
    });
}

main();
```

## 重點總結

### gRPC 核心概念

1. **四種 RPC 類型**
   - Unary（一元）：一個請求，一個響應
   - Server Streaming：一個請求，多個響應
   - Client Streaming：多個請求，一個響應
   - Bidirectional Streaming：多個請求，多個響應

2. **Protocol Buffers 優勢**
   - 比 JSON 更小、更快
   - 強類型
   - 跨語言
   - 向後兼容

3. **性能優勢**
   - HTTP/2 多路復用
   - 二進制序列化
   - 頭部壓縮

### gRPC vs REST 性能對比

| 指標 | REST (JSON) | gRPC (Protobuf) |
|------|-------------|-----------------|
| 消息大小 | ~100% | ~30% |
| 序列化速度 | 慢 | 快 |
| 類型安全 | 無 | 有 |
| 流式支持 | 有限 | 完整 |
| 瀏覽器支持 | 好 | 需要 gRPC-Web |

### 最佳實踐

1. **使用 gRPC 的場景**
   - 微服務間通信
   - 高性能要求
   - 實時雙向通信
   - 多語言環境

2. **不適合 gRPC 的場景**
   - 需要瀏覽器直接訪問
   - 簡單的 CRUD API
   - 調試和測試要求高

3. **開發建議**
   - 使用 Makefile 管理 proto 編譯
   - 添加適當的攔截器
   - 實現健康檢查
   - 使用反射服務便於調試

## 練習題

### 練習 1: 實現認證攔截器

創建一個攔截器驗證客戶端的 API token。

```go
func AuthInterceptor() grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
        // 從 metadata 中獲取 token
        // 驗證 token
        // 調用 handler
    }
}
```

### 練習 2: 實現限流

使用 golang.org/x/time/rate 實現請求限流。

### 練習 3: 添加錯誤碼

定義自定義錯誤碼並在響應中返回。

```protobuf
message ErrorResponse {
  int32 code = 1;
  string message = 2;
  map<string, string> details = 3;
}
```

### 練習 4: 實現重試機制

為 gRPC 客戶端添加自動重試功能。

### 練習 5: gRPC Gateway

集成 grpc-gateway 使 gRPC 服務同時支持 HTTP/JSON。

---

下一章：[Chapter 54: WebSocket 實時通信](54-websocket.md)
