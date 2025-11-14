# Chapter 44: MongoDB 與 Go

## 概述

MongoDB 是最流行的 NoSQL 文檔數據庫。Go 通過官方的 `mongo-driver` 提供了完整的 MongoDB 支持。本章將對比 Mongoose（Node.js）和 mongo-driver（Go）的使用方式。

## Node.js vs Go 對比

### Mongoose (Node.js)
```javascript
const mongoose = require('mongoose');

// 連接
await mongoose.connect('mongodb://localhost:27017/mydb');

// 定義 Schema
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, unique: true },
  age: Number,
  createdAt: { type: Date, default: Date.now }
});

const User = mongoose.model('User', userSchema);

// CRUD 操作
const user = await User.create({ name: 'Alice', email: 'alice@example.com' });
const users = await User.find({ age: { $gt: 18 } });
await User.updateOne({ _id: user._id }, { $set: { age: 26 } });
await User.deleteOne({ _id: user._id });
```

### mongo-driver (Go)
```go
import (
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

// 連接
client, err := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))
db := client.Database("mydb")
collection := db.Collection("users")

// 定義結構（無需 Schema）
type User struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Name      string             `bson:"name"`
    Email     string             `bson:"email"`
    Age       int                `bson:"age,omitempty"`
    CreatedAt time.Time          `bson:"createdAt"`
}

// CRUD 操作
result, err := collection.InsertOne(ctx, User{Name: "Alice", Email: "alice@example.com"})
cursor, err := collection.Find(ctx, bson.M{"age": bson.M{"$gt": 18}})
_, err = collection.UpdateOne(ctx, bson.M{"_id": id}, bson.M{"$set": bson.M{"age": 26}})
_, err = collection.DeleteOne(ctx, bson.M{"_id": id})
```

## 安裝

```bash
go get go.mongodb.org/mongo-driver/mongo
go get go.mongodb.org/mongo-driver/bson
```

## 1. 連接與配置

### 基礎連接

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
    "go.mongodb.org/mongo-driver/mongo/readpref"
)

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // 基礎連接
    client, err := mongo.Connect(ctx, options.Client().
        ApplyURI("mongodb://localhost:27017"))
    if err != nil {
        log.Fatal(err)
    }
    defer client.Disconnect(ctx)

    // 測試連接
    err = client.Ping(ctx, readpref.Primary())
    if err != nil {
        log.Fatal("無法連接到 MongoDB:", err)
    }

    fmt.Println("成功連接到 MongoDB!")

    // 獲取數據庫和集合
    db := client.Database("mydb")
    collection := db.Collection("users")

    fmt.Println("數據庫:", db.Name())
    fmt.Println("集合:", collection.Name())
}
```

### 高級連接配置

```go
package main

import (
    "context"
    "time"

    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func advancedConnection() (*mongo.Client, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    // 連接選項
    clientOptions := options.Client().
        ApplyURI("mongodb://localhost:27017").
        SetAuth(options.Credential{
            Username: "admin",
            Password: "password",
        }).
        SetMaxPoolSize(50).              // 連接池大小
        SetMinPoolSize(10).              // 最小連接數
        SetMaxConnIdleTime(30 * time.Second). // 連接空閒時間
        SetServerSelectionTimeout(5 * time.Second). // 服務器選擇超時
        SetSocketTimeout(10 * time.Second). // Socket 超時
        SetCompressors([]string{"snappy", "zlib"}). // 壓縮
        SetRetryWrites(true).            // 自動重試寫入
        SetRetryReads(true)              // 自動重試讀取

    // 連接
    client, err := mongo.Connect(ctx, clientOptions)
    if err != nil {
        return nil, err
    }

    // 測試連接
    if err := client.Ping(ctx, nil); err != nil {
        return nil, err
    }

    return client, nil
}
```

**Node.js 對比：**
```javascript
const mongoose = require('mongoose');

// 基礎連接
await mongoose.connect('mongodb://localhost:27017/mydb');

// 高級配置
await mongoose.connect('mongodb://localhost:27017/mydb', {
  auth: {
    username: 'admin',
    password: 'password'
  },
  maxPoolSize: 50,
  minPoolSize: 10,
  socketTimeoutMS: 10000,
  serverSelectionTimeoutMS: 5000,
  retryWrites: true,
  compressors: ['snappy', 'zlib']
});

console.log('成功連接到 MongoDB!');
```

## 2. 文檔模型定義

### 基礎模型

```go
package main

import (
    "time"

    "go.mongodb.org/mongo-driver/bson/primitive"
)

// User 模型
type User struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Name      string             `bson:"name"`
    Email     string             `bson:"email"`
    Age       int                `bson:"age,omitempty"`
    Active    bool               `bson:"active"`
    Tags      []string           `bson:"tags,omitempty"`
    CreatedAt time.Time          `bson:"createdAt"`
    UpdatedAt time.Time          `bson:"updatedAt"`
}

// 嵌套文檔
type Address struct {
    Street  string `bson:"street"`
    City    string `bson:"city"`
    Country string `bson:"country"`
    ZipCode string `bson:"zipCode"`
}

type UserWithAddress struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Name      string             `bson:"name"`
    Email     string             `bson:"email"`
    Address   Address            `bson:"address"`
    CreatedAt time.Time          `bson:"createdAt"`
}

// 數組嵌套
type Post struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Title     string             `bson:"title"`
    Content   string             `bson:"content"`
    Author    primitive.ObjectID `bson:"author"`
    Comments  []Comment          `bson:"comments"`
    Tags      []string           `bson:"tags"`
    CreatedAt time.Time          `bson:"createdAt"`
}

type Comment struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Author    string             `bson:"author"`
    Content   string             `bson:"content"`
    CreatedAt time.Time          `bson:"createdAt"`
}

// BSON 標籤說明
type Example struct {
    ID       primitive.ObjectID `bson:"_id,omitempty"` // MongoDB ObjectID
    Name     string             `bson:"name"`          // 字段名映射
    Age      int                `bson:"age,omitempty"` // 零值時忽略
    Internal string             `bson:"-"`             // 忽略此字段
    Inline   Address            `bson:",inline"`       // 內聯嵌入
}
```

**Node.js 對比：**
```javascript
const mongoose = require('mongoose');

// User Schema
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  age: Number,
  active: { type: Boolean, default: true },
  tags: [String],
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
});

const User = mongoose.model('User', userSchema);

// 嵌套文檔
const addressSchema = new mongoose.Schema({
  street: String,
  city: String,
  country: String,
  zipCode: String
});

const userWithAddressSchema = new mongoose.Schema({
  name: String,
  email: String,
  address: addressSchema
});

// 數組嵌套
const commentSchema = new mongoose.Schema({
  author: String,
  content: String,
  createdAt: { type: Date, default: Date.now }
});

const postSchema = new mongoose.Schema({
  title: String,
  content: String,
  author: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  comments: [commentSchema],
  tags: [String]
});
```

## 3. CRUD 操作

### Create - 插入文檔

```go
package main

import (
    "context"
    "fmt"
    "time"

    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
)

func createExamples(collection *mongo.Collection) {
    ctx := context.Background()

    // 1. 插入單個文檔
    user := User{
        Name:      "Alice",
        Email:     "alice@example.com",
        Age:       25,
        Active:    true,
        Tags:      []string{"developer", "golang"},
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    result, err := collection.InsertOne(ctx, user)
    if err != nil {
        panic(err)
    }

    insertedID := result.InsertedID.(primitive.ObjectID)
    fmt.Println("插入文檔 ID:", insertedID.Hex())

    // 2. 插入多個文檔
    users := []interface{}{
        User{Name: "Bob", Email: "bob@example.com", Age: 30, CreatedAt: time.Now()},
        User{Name: "Charlie", Email: "charlie@example.com", Age: 35, CreatedAt: time.Now()},
    }

    manyResult, err := collection.InsertMany(ctx, users)
    if err != nil {
        panic(err)
    }

    fmt.Printf("插入 %d 個文檔\n", len(manyResult.InsertedIDs))
    for i, id := range manyResult.InsertedIDs {
        fmt.Printf("文檔 %d ID: %s\n", i+1, id.(primitive.ObjectID).Hex())
    }

    // 3. 插入時設置選項
    opts := options.InsertOne().SetBypassDocumentValidation(true)
    result, err = collection.InsertOne(ctx, user, opts)
}
```

**Node.js 對比：**
```javascript
// 插入單個
const user = await User.create({
  name: 'Alice',
  email: 'alice@example.com',
  age: 25,
  active: true,
  tags: ['developer', 'golang']
});
console.log('插入文檔 ID:', user._id);

// 插入多個
const users = await User.insertMany([
  { name: 'Bob', email: 'bob@example.com', age: 30 },
  { name: 'Charlie', email: 'charlie@example.com', age: 35 }
]);
console.log(`插入 ${users.length} 個文檔`);
```

### Read - 查詢文檔

```go
package main

import (
    "context"
    "fmt"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func readExamples(collection *mongo.Collection) {
    ctx := context.Background()

    // 1. 查詢單個文檔
    var user User
    err := collection.FindOne(ctx, bson.M{"email": "alice@example.com"}).Decode(&user)
    if err == mongo.ErrNoDocuments {
        fmt.Println("文檔不存在")
    } else if err != nil {
        panic(err)
    }
    fmt.Printf("找到用戶: %+v\n", user)

    // 2. 按 ID 查詢
    id, _ := primitive.ObjectIDFromHex("507f1f77bcf86cd799439011")
    err = collection.FindOne(ctx, bson.M{"_id": id}).Decode(&user)

    // 3. 查詢多個文檔
    cursor, err := collection.Find(ctx, bson.M{"age": bson.M{"$gt": 18}})
    if err != nil {
        panic(err)
    }
    defer cursor.Close(ctx)

    var users []User
    if err = cursor.All(ctx, &users); err != nil {
        panic(err)
    }
    fmt.Printf("找到 %d 個用戶\n", len(users))

    // 4. 迭代查詢結果
    cursor, err = collection.Find(ctx, bson.M{})
    defer cursor.Close(ctx)

    for cursor.Next(ctx) {
        var u User
        if err := cursor.Decode(&u); err != nil {
            panic(err)
        }
        fmt.Printf("用戶: %s\n", u.Name)
    }

    // 5. 複雜查詢條件
    filter := bson.M{
        "age": bson.M{"$gte": 18, "$lte": 30},
        "active": true,
        "tags": bson.M{"$in": []string{"developer", "designer"}},
        "$or": []bson.M{
            {"name": bson.M{"$regex": "^A"}},
            {"email": bson.M{"$regex": "@gmail.com$"}},
        },
    }
    cursor, err = collection.Find(ctx, filter)

    // 6. 投影（選擇字段）
    opts := options.Find().SetProjection(bson.M{
        "name": 1,
        "email": 1,
        "_id": 0,
    })
    cursor, err = collection.Find(ctx, bson.M{}, opts)

    // 7. 排序
    opts = options.Find().
        SetSort(bson.D{{"age", -1}, {"name", 1}}) // age 降序，name 升序
    cursor, err = collection.Find(ctx, bson.M{}, opts)

    // 8. 分頁
    page := 1
    pageSize := 10
    opts = options.Find().
        SetLimit(int64(pageSize)).
        SetSkip(int64((page - 1) * pageSize))
    cursor, err = collection.Find(ctx, bson.M{}, opts)

    // 9. 計數
    count, err := collection.CountDocuments(ctx, bson.M{"age": bson.M{"$gt": 18}})
    fmt.Printf("符合條件的文檔數: %d\n", count)

    // 10. 聚合查詢
    pipeline := mongo.Pipeline{
        bson.D{{"$match", bson.M{"active": true}}},
        bson.D{{"$group", bson.M{
            "_id": "$age",
            "count": bson.M{"$sum": 1},
            "avgAge": bson.M{"$avg": "$age"},
        }}},
        bson.D{{"$sort", bson.M{"count": -1}}},
    }
    cursor, err = collection.Aggregate(ctx, pipeline)
}
```

**Node.js 對比：**
```javascript
// 查詢單個
const user = await User.findOne({ email: 'alice@example.com' });

// 按 ID 查詢
const user = await User.findById('507f1f77bcf86cd799439011');

// 查詢多個
const users = await User.find({ age: { $gt: 18 } });

// 複雜查詢
const users = await User.find({
  age: { $gte: 18, $lte: 30 },
  active: true,
  tags: { $in: ['developer', 'designer'] },
  $or: [
    { name: /^A/ },
    { email: /@gmail.com$/ }
  ]
});

// 投影
const users = await User.find({}, 'name email -_id');

// 排序
const users = await User.find().sort({ age: -1, name: 1 });

// 分頁
const page = 1;
const pageSize = 10;
const users = await User.find()
  .limit(pageSize)
  .skip((page - 1) * pageSize);

// 計數
const count = await User.countDocuments({ age: { $gt: 18 } });

// 聚合
const results = await User.aggregate([
  { $match: { active: true } },
  { $group: { _id: '$age', count: { $sum: 1 }, avgAge: { $avg: '$age' } } },
  { $sort: { count: -1 } }
]);
```

### Update - 更新文檔

```go
package main

import (
    "context"
    "fmt"
    "time"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func updateExamples(collection *mongo.Collection) {
    ctx := context.Background()

    // 1. 更新單個文檔
    filter := bson.M{"email": "alice@example.com"}
    update := bson.M{
        "$set": bson.M{
            "age": 26,
            "updatedAt": time.Now(),
        },
    }
    result, err := collection.UpdateOne(ctx, filter, update)
    if err != nil {
        panic(err)
    }
    fmt.Printf("匹配: %d, 修改: %d\n", result.MatchedCount, result.ModifiedCount)

    // 2. 更新多個文檔
    filter = bson.M{"age": bson.M{"$lt": 18}}
    update = bson.M{"$set": bson.M{"active": false}}
    result, err = collection.UpdateMany(ctx, filter, update)

    // 3. Upsert（不存在則插入）
    opts := options.Update().SetUpsert(true)
    filter = bson.M{"email": "new@example.com"}
    update = bson.M{
        "$set": bson.M{
            "name": "New User",
            "age": 20,
        },
    }
    result, err = collection.UpdateOne(ctx, filter, update, opts)
    if result.UpsertedCount > 0 {
        fmt.Println("插入新文檔 ID:", result.UpsertedID)
    }

    // 4. 更新操作符
    update = bson.M{
        "$inc": bson.M{"age": 1},                    // 增加
        "$mul": bson.M{"score": 1.1},                // 乘以
        "$rename": bson.M{"name": "fullName"},       // 重命名字段
        "$unset": bson.M{"oldField": ""},            // 刪除字段
        "$set": bson.M{"status": "active"},          // 設置值
        "$currentDate": bson.M{"lastModified": true}, // 當前日期
    }
    collection.UpdateOne(ctx, filter, update)

    // 5. 數組操作
    // 添加到數組
    update = bson.M{"$push": bson.M{"tags": "newTag"}}
    collection.UpdateOne(ctx, filter, update)

    // 添加多個到數組
    update = bson.M{"$push": bson.M{"tags": bson.M{"$each": []string{"tag1", "tag2"}}}}
    collection.UpdateOne(ctx, filter, update)

    // 從數組刪除
    update = bson.M{"$pull": bson.M{"tags": "oldTag"}}
    collection.UpdateOne(ctx, filter, update)

    // 添加到集合（不重複）
    update = bson.M{"$addToSet": bson.M{"tags": "uniqueTag"}}
    collection.UpdateOne(ctx, filter, update)

    // 彈出數組元素
    update = bson.M{"$pop": bson.M{"tags": 1}}  // 1: 最後一個, -1: 第一個
    collection.UpdateOne(ctx, filter, update)

    // 6. FindOneAndUpdate（返回更新後的文檔）
    opts2 := options.FindOneAndUpdate().
        SetReturnDocument(options.After). // 返回更新後的文檔
        SetUpsert(true)

    var updatedUser User
    err = collection.FindOneAndUpdate(
        ctx,
        bson.M{"email": "alice@example.com"},
        bson.M{"$set": bson.M{"age": 27}},
        opts2,
    ).Decode(&updatedUser)

    fmt.Printf("更新後的用戶: %+v\n", updatedUser)
}
```

**Node.js 對比：**
```javascript
// 更新單個
const result = await User.updateOne(
  { email: 'alice@example.com' },
  { $set: { age: 26, updatedAt: new Date() } }
);
console.log(`匹配: ${result.matchedCount}, 修改: ${result.modifiedCount}`);

// 更新多個
await User.updateMany(
  { age: { $lt: 18 } },
  { $set: { active: false } }
);

// Upsert
await User.updateOne(
  { email: 'new@example.com' },
  { $set: { name: 'New User', age: 20 } },
  { upsert: true }
);

// 數組操作
await User.updateOne(
  { email: 'alice@example.com' },
  { $push: { tags: 'newTag' } }
);

// findOneAndUpdate
const updatedUser = await User.findOneAndUpdate(
  { email: 'alice@example.com' },
  { $set: { age: 27 } },
  { new: true, upsert: true }
);
```

### Delete - 刪除文檔

```go
package main

import (
    "context"
    "fmt"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
)

func deleteExamples(collection *mongo.Collection) {
    ctx := context.Background()

    // 1. 刪除單個文檔
    result, err := collection.DeleteOne(ctx, bson.M{"email": "alice@example.com"})
    if err != nil {
        panic(err)
    }
    fmt.Printf("刪除 %d 個文檔\n", result.DeletedCount)

    // 2. 刪除多個文檔
    result, err = collection.DeleteMany(ctx, bson.M{"age": bson.M{"$lt": 18}})
    fmt.Printf("刪除 %d 個文檔\n", result.DeletedCount)

    // 3. 刪除所有文檔
    result, err = collection.DeleteMany(ctx, bson.M{})

    // 4. FindOneAndDelete（返回被刪除的文檔）
    var deletedUser User
    err = collection.FindOneAndDelete(
        ctx,
        bson.M{"email": "bob@example.com"},
    ).Decode(&deletedUser)

    if err == mongo.ErrNoDocuments {
        fmt.Println("文檔不存在")
    } else {
        fmt.Printf("刪除的用戶: %+v\n", deletedUser)
    }
}
```

**Node.js 對比：**
```javascript
// 刪除單個
const result = await User.deleteOne({ email: 'alice@example.com' });
console.log(`刪除 ${result.deletedCount} 個文檔`);

// 刪除多個
await User.deleteMany({ age: { $lt: 18 } });

// findOneAndDelete
const deletedUser = await User.findOneAndDelete({ email: 'bob@example.com' });
console.log('刪除的用戶:', deletedUser);
```

## 4. 聚合管道

```go
package main

import (
    "context"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
)

func aggregationExamples(collection *mongo.Collection) {
    ctx := context.Background()

    // 1. 基礎聚合
    pipeline := mongo.Pipeline{
        // Match - 過濾
        bson.D{{"$match", bson.M{"active": true}}},

        // Group - 分組
        bson.D{{"$group", bson.M{
            "_id": "$age",
            "count": bson.M{"$sum": 1},
            "avgAge": bson.M{"$avg": "$age"},
            "users": bson.M{"$push": "$name"},
        }}},

        // Sort - 排序
        bson.D{{"$sort", bson.M{"count": -1}}},

        // Limit - 限制
        bson.D{{"$limit", 10}},
    }

    cursor, err := collection.Aggregate(ctx, pipeline)
    if err != nil {
        panic(err)
    }
    defer cursor.Close(ctx)

    var results []bson.M
    if err = cursor.All(ctx, &results); err != nil {
        panic(err)
    }

    // 2. 複雜聚合
    pipeline = mongo.Pipeline{
        // Unwind - 展開數組
        bson.D{{"$unwind", "$tags"}},

        // Group
        bson.D{{"$group", bson.M{
            "_id": "$tags",
            "count": bson.M{"$sum": 1},
        }}},

        // Project - 投影
        bson.D{{"$project", bson.M{
            "tag": "$_id",
            "count": 1,
            "_id": 0,
        }}},

        // Sort
        bson.D{{"$sort", bson.M{"count": -1}}},
    }

    cursor, err = collection.Aggregate(ctx, pipeline)

    // 3. Lookup - 關聯查詢（類似 JOIN）
    pipeline = mongo.Pipeline{
        bson.D{{"$lookup", bson.M{
            "from": "posts",
            "localField": "_id",
            "foreignField": "author",
            "as": "userPosts",
        }}},

        bson.D{{"$project", bson.M{
            "name": 1,
            "email": 1,
            "postCount": bson.M{"$size": "$userPosts"},
        }}},
    }

    // 4. Facet - 多個聚合
    pipeline = mongo.Pipeline{
        bson.D{{"$facet", bson.M{
            "ageGroups": []bson.M{
                {"$group": bson.M{"_id": "$age", "count": bson.M{"$sum": 1}}},
                {"$sort": bson.M{"count": -1}},
            },
            "activeUsers": []bson.M{
                {"$match": bson.M{"active": true}},
                {"$count": "total"},
            },
        }}},
    }

    // 5. 統計聚合
    pipeline = mongo.Pipeline{
        bson.D{{"$group", bson.M{
            "_id": nil,
            "totalUsers": bson.M{"$sum": 1},
            "avgAge": bson.M{"$avg": "$age"},
            "minAge": bson.M{"$min": "$age"},
            "maxAge": bson.M{"$max": "$age"},
            "stdDevAge": bson.M{"$stdDevPop": "$age"},
        }}},
    }
}
```

**Node.js 對比：**
```javascript
// 基礎聚合
const results = await User.aggregate([
  { $match: { active: true } },
  { $group: {
    _id: '$age',
    count: { $sum: 1 },
    avgAge: { $avg: '$age' },
    users: { $push: '$name' }
  }},
  { $sort: { count: -1 } },
  { $limit: 10 }
]);

// Lookup（關聯查詢）
const results = await User.aggregate([
  { $lookup: {
    from: 'posts',
    localField: '_id',
    foreignField: 'author',
    as: 'userPosts'
  }},
  { $project: {
    name: 1,
    email: 1,
    postCount: { $size: '$userPosts' }
  }}
]);
```

## 5. 索引

```go
package main

import (
    "context"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

func indexExamples(collection *mongo.Collection) {
    ctx := context.Background()

    // 1. 創建單字段索引
    indexModel := mongo.IndexModel{
        Keys: bson.D{{"email", 1}}, // 1: 升序, -1: 降序
        Options: options.Index().
            SetUnique(true).
            SetName("email_unique_index"),
    }
    name, err := collection.Indexes().CreateOne(ctx, indexModel)
    fmt.Println("創建索引:", name)

    // 2. 複合索引
    indexModel = mongo.IndexModel{
        Keys: bson.D{
            {"age", 1},
            {"name", 1},
        },
    }
    collection.Indexes().CreateOne(ctx, indexModel)

    // 3. 文本索引（全文搜索）
    indexModel = mongo.IndexModel{
        Keys: bson.D{
            {"name", "text"},
            {"bio", "text"},
        },
    }
    collection.Indexes().CreateOne(ctx, indexModel)

    // 使用文本索引搜索
    filter := bson.M{"$text": bson.M{"$search": "developer golang"}}
    cursor, err := collection.Find(ctx, filter)

    // 4. TTL 索引（自動過期）
    indexModel = mongo.IndexModel{
        Keys: bson.D{{"createdAt", 1}},
        Options: options.Index().
            SetExpireAfterSeconds(3600), // 1小時後過期
    }
    collection.Indexes().CreateOne(ctx, indexModel)

    // 5. 列出所有索引
    cursor, err := collection.Indexes().List(ctx)
    defer cursor.Close(ctx)

    var indexes []bson.M
    cursor.All(ctx, &indexes)
    for _, index := range indexes {
        fmt.Printf("索引: %+v\n", index)
    }

    // 6. 刪除索引
    _, err = collection.Indexes().DropOne(ctx, "email_unique_index")

    // 7. 刪除所有索引（除了 _id）
    _, err = collection.Indexes().DropAll(ctx)
}
```

**Node.js 對比：**
```javascript
// 創建索引
await User.createIndex({ email: 1 }, { unique: true, name: 'email_unique_index' });

// 複合索引
await User.createIndex({ age: 1, name: 1 });

// 文本索引
await User.createIndex({ name: 'text', bio: 'text' });

// 搜索
const users = await User.find({ $text: { $search: 'developer golang' } });

// TTL 索引
await Session.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 });

// 列出索引
const indexes = await User.collection.indexes();

// 刪除索引
await User.collection.dropIndex('email_unique_index');
```

## 完整示例：博客系統

```go
package main

import (
    "context"
    "fmt"
    "time"

    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

// 模型
type User struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Username  string             `bson:"username"`
    Email     string             `bson:"email"`
    Password  string             `bson:"password"`
    CreatedAt time.Time          `bson:"createdAt"`
}

type Post struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Title     string             `bson:"title"`
    Content   string             `bson:"content"`
    Author    primitive.ObjectID `bson:"author"`
    Tags      []string           `bson:"tags"`
    Views     int                `bson:"views"`
    Comments  []Comment          `bson:"comments"`
    CreatedAt time.Time          `bson:"createdAt"`
    UpdatedAt time.Time          `bson:"updatedAt"`
}

type Comment struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Author    string             `bson:"author"`
    Content   string             `bson:"content"`
    CreatedAt time.Time          `bson:"createdAt"`
}

// Repository
type BlogRepository struct {
    db             *mongo.Database
    usersCol       *mongo.Collection
    postsCol       *mongo.Collection
}

func NewBlogRepository(client *mongo.Client) *BlogRepository {
    db := client.Database("blog")
    return &BlogRepository{
        db:       db,
        usersCol: db.Collection("users"),
        postsCol: db.Collection("posts"),
    }
}

// 創建用戶
func (r *BlogRepository) CreateUser(ctx context.Context, username, email, password string) (*User, error) {
    user := &User{
        Username:  username,
        Email:     email,
        Password:  password, // 實際應用中應該加密
        CreatedAt: time.Now(),
    }

    result, err := r.usersCol.InsertOne(ctx, user)
    if err != nil {
        return nil, err
    }

    user.ID = result.InsertedID.(primitive.ObjectID)
    return user, nil
}

// 創建文章
func (r *BlogRepository) CreatePost(ctx context.Context, title, content string, authorID primitive.ObjectID, tags []string) (*Post, error) {
    post := &Post{
        Title:     title,
        Content:   content,
        Author:    authorID,
        Tags:      tags,
        Views:     0,
        Comments:  []Comment{},
        CreatedAt: time.Now(),
        UpdatedAt: time.Now(),
    }

    result, err := r.postsCol.InsertOne(ctx, post)
    if err != nil {
        return nil, err
    }

    post.ID = result.InsertedID.(primitive.ObjectID)
    return post, nil
}

// 獲取文章列表（帶分頁）
func (r *BlogRepository) GetPosts(ctx context.Context, page, pageSize int) ([]Post, int64, error) {
    // 計數
    total, err := r.postsCol.CountDocuments(ctx, bson.M{})
    if err != nil {
        return nil, 0, err
    }

    // 查詢
    opts := options.Find().
        SetSort(bson.D{{"createdAt", -1}}).
        SetLimit(int64(pageSize)).
        SetSkip(int64((page - 1) * pageSize))

    cursor, err := r.postsCol.Find(ctx, bson.M{}, opts)
    if err != nil {
        return nil, 0, err
    }
    defer cursor.Close(ctx)

    var posts []Post
    if err = cursor.All(ctx, &posts); err != nil {
        return nil, 0, err
    }

    return posts, total, nil
}

// 搜索文章
func (r *BlogRepository) SearchPosts(ctx context.Context, keyword string, tags []string) ([]Post, error) {
    filter := bson.M{}

    if keyword != "" {
        filter["$or"] = []bson.M{
            {"title": bson.M{"$regex": keyword, "$options": "i"}},
            {"content": bson.M{"$regex": keyword, "$options": "i"}},
        }
    }

    if len(tags) > 0 {
        filter["tags"] = bson.M{"$in": tags}
    }

    cursor, err := r.postsCol.Find(ctx, filter)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(ctx)

    var posts []Post
    if err = cursor.All(ctx, &posts); err != nil {
        return nil, err
    }

    return posts, nil
}

// 添加評論
func (r *BlogRepository) AddComment(ctx context.Context, postID primitive.ObjectID, author, content string) error {
    comment := Comment{
        ID:        primitive.NewObjectID(),
        Author:    author,
        Content:   content,
        CreatedAt: time.Now(),
    }

    update := bson.M{
        "$push": bson.M{"comments": comment},
        "$set": bson.M{"updatedAt": time.Now()},
    }

    _, err := r.postsCol.UpdateOne(ctx, bson.M{"_id": postID}, update)
    return err
}

// 增加閱讀數
func (r *BlogRepository) IncrementViews(ctx context.Context, postID primitive.ObjectID) error {
    _, err := r.postsCol.UpdateOne(
        ctx,
        bson.M{"_id": postID},
        bson.M{"$inc": bson.M{"views": 1}},
    )
    return err
}

// 獲取統計信息
func (r *BlogRepository) GetStats(ctx context.Context) (map[string]interface{}, error) {
    stats := make(map[string]interface{})

    // 用戶總數
    userCount, _ := r.usersCol.CountDocuments(ctx, bson.M{})
    stats["totalUsers"] = userCount

    // 文章總數
    postCount, _ := r.postsCol.CountDocuments(ctx, bson.M{})
    stats["totalPosts"] = postCount

    // 標籤統計
    pipeline := mongo.Pipeline{
        bson.D{{"$unwind", "$tags"}},
        bson.D{{"$group", bson.M{
            "_id": "$tags",
            "count": bson.M{"$sum": 1},
        }}},
        bson.D{{"$sort", bson.M{"count": -1}}},
        bson.D{{"$limit", 10}},
    }

    cursor, err := r.postsCol.Aggregate(ctx, pipeline)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(ctx)

    var tagStats []bson.M
    cursor.All(ctx, &tagStats)
    stats["topTags"] = tagStats

    return stats, nil
}

func main() {
    ctx := context.Background()

    // 連接 MongoDB
    client, err := mongo.Connect(ctx, options.Client().
        ApplyURI("mongodb://localhost:27017"))
    if err != nil {
        panic(err)
    }
    defer client.Disconnect(ctx)

    // 創建 Repository
    repo := NewBlogRepository(client)

    // 創建用戶
    user, _ := repo.CreateUser(ctx, "alice", "alice@example.com", "password123")
    fmt.Printf("創建用戶: %s (ID: %s)\n", user.Username, user.ID.Hex())

    // 創建文章
    post, _ := repo.CreatePost(ctx, "MongoDB with Go", "This is a tutorial...",
        user.ID, []string{"go", "mongodb", "tutorial"})
    fmt.Printf("創建文章: %s (ID: %s)\n", post.Title, post.ID.Hex())

    // 添加評論
    repo.AddComment(ctx, post.ID, "Bob", "Great article!")

    // 獲取文章列表
    posts, total, _ := repo.GetPosts(ctx, 1, 10)
    fmt.Printf("文章列表: %d/%d\n", len(posts), total)

    // 搜索文章
    searchResults, _ := repo.SearchPosts(ctx, "MongoDB", []string{"go"})
    fmt.Printf("搜索結果: %d 篇\n", len(searchResults))

    // 獲取統計
    stats, _ := repo.GetStats(ctx)
    fmt.Printf("統計信息: %+v\n", stats)
}
```

## 重點總結

### MongoDB vs SQL

| 特性 | MongoDB | SQL |
|------|---------|-----|
| **數據模型** | 文檔（JSON） | 表格 |
| **Schema** | 動態 | 固定 |
| **關聯** | 嵌入/引用 | JOIN |
| **擴展性** | 水平擴展 | 垂直擴展 |
| **查詢** | 文檔查詢 | SQL |

### 最佳實踐

1. **合理使用嵌入 vs 引用**
2. **為常用查詢創建索引**
3. **使用聚合管道處理複雜查詢**
4. **注意文檔大小限制（16MB）**
5. **使用 Context 控制超時**

## 練習題

1. **基礎**：實現一個待辦事項系統（CRUD）
2. **中級**：實現一個社交媒體系統（用戶、帖子、評論、點讚）
3. **高級**：實現一個電商系統（商品、訂單、庫存管理）
4. **實戰**：構建一個完整的博客系統，包含搜索、標籤、評論功能

## 下一步

下一章將學習 Redis 與 Go 的集成，探索緩存和會話管理。
