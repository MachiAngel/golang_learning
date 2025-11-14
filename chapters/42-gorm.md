# Chapter 42: GORM ORM

## 概述

GORM 是 Go 中最流行的 ORM（Object-Relational Mapping）庫，提供了類似於 Sequelize（Node.js）或 TypeORM 的功能。它簡化了數據庫操作，提供了模型定義、自動遷移、關聯、鉤子等功能。

## Node.js vs Go 對比

### Sequelize (Node.js)
```javascript
// Node.js - Sequelize
const { Sequelize, DataTypes } = require('sequelize');

const sequelize = new Sequelize('mysql://root:password@localhost:3306/mydb');

const User = sequelize.define('User', {
  name: {
    type: DataTypes.STRING,
    allowNull: false
  },
  email: {
    type: DataTypes.STRING,
    unique: true
  },
  age: DataTypes.INTEGER
});

// 同步模型
await sequelize.sync();

// 創建記錄
const user = await User.create({
  name: 'Alice',
  email: 'alice@example.com',
  age: 25
});

// 查詢
const users = await User.findAll({ where: { age: { [Op.gt]: 18 } } });
```

### GORM (Go)
```go
// Go - GORM
import (
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

type User struct {
    ID    uint   `gorm:"primaryKey"`
    Name  string `gorm:"not null"`
    Email string `gorm:"unique"`
    Age   int
}

// 連接數據庫
dsn := "root:password@tcp(127.0.0.1:3306)/mydb?parseTime=true"
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})

// 自動遷移
db.AutoMigrate(&User{})

// 創建記錄
user := User{Name: "Alice", Email: "alice@example.com", Age: 25}
db.Create(&user)

// 查詢
var users []User
db.Where("age > ?", 18).Find(&users)
```

## 安裝與設置

```bash
# 安裝 GORM
go get -u gorm.io/gorm

# 安裝數據庫驅動
go get -u gorm.io/driver/mysql
go get -u gorm.io/driver/postgres
go get -u gorm.io/driver/sqlite
```

## 1. 模型定義

### 基礎模型

```go
package main

import (
    "time"
    "gorm.io/gorm"
)

// 基礎模型定義
type User struct {
    ID        uint           `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`  // 軟刪除
    Name      string         `gorm:"size:100;not null"`
    Email     string         `gorm:"uniqueIndex;size:100"`
    Age       int            `gorm:"default:0"`
    Active    bool           `gorm:"default:true"`
}

// 使用嵌入的 gorm.Model（包含 ID, CreatedAt, UpdatedAt, DeletedAt）
type Product struct {
    gorm.Model
    Code  string  `gorm:"uniqueIndex"`
    Price float64 `gorm:"precision:10;scale:2"`
    Stock int     `gorm:"default:0;check:stock >= 0"`
}

// 自定義表名
func (User) TableName() string {
    return "my_users"
}
```

**Node.js 對比：**
```javascript
// Sequelize
const User = sequelize.define('User', {
  id: {
    type: DataTypes.INTEGER,
    primaryKey: true,
    autoIncrement: true
  },
  name: {
    type: DataTypes.STRING(100),
    allowNull: false
  },
  email: {
    type: DataTypes.STRING(100),
    unique: true
  },
  age: {
    type: DataTypes.INTEGER,
    defaultValue: 0
  },
  active: {
    type: DataTypes.BOOLEAN,
    defaultValue: true
  }
}, {
  timestamps: true,
  paranoid: true,  // 軟刪除
  tableName: 'my_users'
});

const Product = sequelize.define('Product', {
  code: {
    type: DataTypes.STRING,
    unique: true
  },
  price: {
    type: DataTypes.DECIMAL(10, 2)
  },
  stock: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    validate: {
      min: 0
    }
  }
});
```

### 標籤說明

```go
type Example struct {
    // 主鍵
    ID uint `gorm:"primaryKey"`

    // 列名
    Name string `gorm:"column:user_name"`

    // 數據類型
    Age int `gorm:"type:int"`

    // 大小
    Email string `gorm:"size:255"`

    // 精度（用於 decimal）
    Price float64 `gorm:"precision:10;scale:2"`

    // 約束
    Username string `gorm:"unique;not null"`

    // 默認值
    Role string `gorm:"default:user"`

    // 索引
    Code string `gorm:"index"`
    Phone string `gorm:"uniqueIndex"`

    // 忽略字段
    Password string `gorm:"-"`

    // 只讀字段
    CreatedAt time.Time `gorm:"<-:create"`

    // 檢查約束
    Age int `gorm:"check:age >= 18"`
}
```

## 2. CRUD 操作

### 創建 (Create)

```go
package main

import (
    "fmt"
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name  string
    Email string
    Age   int
}

func main() {
    db, _ := gorm.Open(mysql.Open("root:password@tcp(localhost:3306)/mydb"))

    // 1. 創建單個記錄
    user := User{Name: "Alice", Email: "alice@example.com", Age: 25}
    result := db.Create(&user)

    fmt.Println("插入記錄數:", result.RowsAffected)
    fmt.Println("新記錄 ID:", user.ID)
    fmt.Println("錯誤:", result.Error)

    // 2. 創建多個記錄
    users := []User{
        {Name: "Bob", Email: "bob@example.com", Age: 30},
        {Name: "Charlie", Email: "charlie@example.com", Age: 35},
    }
    db.Create(&users)

    // 3. 指定字段創建
    db.Select("Name", "Age").Create(&User{
        Name: "David",
        Email: "david@example.com",  // 會被忽略
        Age: 40,
    })

    // 4. 忽略某些字段
    db.Omit("Age").Create(&User{
        Name: "Eve",
        Email: "eve@example.com",
        Age: 45,  // 會被忽略，使用默認值
    })

    // 5. 批量創建（分批）
    var manyUsers []User
    for i := 0; i < 1000; i++ {
        manyUsers = append(manyUsers, User{
            Name: fmt.Sprintf("User%d", i),
            Email: fmt.Sprintf("user%d@example.com", i),
        })
    }
    // 每批 100 條
    db.CreateInBatches(manyUsers, 100)
}
```

**Node.js 對比：**
```javascript
// Sequelize
// 1. 創建單個記錄
const user = await User.create({
  name: 'Alice',
  email: 'alice@example.com',
  age: 25
});
console.log('新記錄 ID:', user.id);

// 2. 批量創建
const users = await User.bulkCreate([
  {name: 'Bob', email: 'bob@example.com', age: 30},
  {name: 'Charlie', email: 'charlie@example.com', age: 35}
]);

// 3. 指定字段
const user2 = await User.create({
  name: 'David',
  email: 'david@example.com',
  age: 40
}, {
  fields: ['name', 'age']  // 只創建這些字段
});
```

### 查詢 (Read)

```go
package main

import (
    "fmt"
    "gorm.io/gorm"
)

func queryExamples(db *gorm.DB) {
    var user User
    var users []User

    // 1. 獲取第一條記錄（按主鍵升序）
    db.First(&user)
    fmt.Println("First:", user)

    // 2. 獲取最後一條記錄（按主鍵降序）
    db.Last(&user)

    // 3. 獲取所有記錄
    db.Find(&users)

    // 4. 按主鍵查詢
    db.First(&user, 10)  // WHERE id = 10
    db.Find(&users, []int{1, 2, 3})  // WHERE id IN (1,2,3)

    // 5. Where 條件
    db.Where("name = ?", "Alice").First(&user)
    db.Where("name <> ?", "Alice").Find(&users)
    db.Where("age > ?", 18).Find(&users)
    db.Where("age BETWEEN ? AND ?", 20, 30).Find(&users)
    db.Where("name IN ?", []string{"Alice", "Bob"}).Find(&users)
    db.Where("name LIKE ?", "%li%").Find(&users)

    // 6. Struct 條件（只會查詢非零值）
    db.Where(&User{Name: "Alice", Age: 25}).Find(&users)

    // 7. Map 條件
    db.Where(map[string]interface{}{
        "name": "Alice",
        "age": 25,
    }).Find(&users)

    // 8. Not 條件
    db.Not("name = ?", "Alice").Find(&users)
    db.Not(map[string]interface{}{"name": []string{"Alice", "Bob"}}).Find(&users)

    // 9. Or 條件
    db.Where("name = ?", "Alice").Or("name = ?", "Bob").Find(&users)

    // 10. 選擇特定字段
    db.Select("name", "age").Find(&users)
    db.Select([]string{"name", "age"}).Find(&users)

    // 11. Order 排序
    db.Order("age desc, name").Find(&users)

    // 12. Limit & Offset
    db.Limit(10).Offset(5).Find(&users)

    // 13. Group & Having
    type Result struct {
        Age   int
        Count int
    }
    var results []Result
    db.Model(&User{}).Select("age, count(*) as count").
        Group("age").Having("count > ?", 2).Scan(&results)

    // 14. 去重
    db.Distinct("name").Find(&users)

    // 15. Joins
    db.Joins("Company").Find(&users)  // 假設有關聯

    // 16. Scan 到自定義結構
    type UserInfo struct {
        Name  string
        Email string
    }
    var userInfos []UserInfo
    db.Model(&User{}).Scan(&userInfos)

    // 17. 檢查記錄是否存在
    result := db.Where("email = ?", "alice@example.com").First(&user)
    if result.Error == gorm.ErrRecordNotFound {
        fmt.Println("記錄不存在")
    }

    // 18. 計數
    var count int64
    db.Model(&User{}).Where("age > ?", 18).Count(&count)
    fmt.Println("符合條件的記錄數:", count)
}
```

**Node.js 對比：**
```javascript
// Sequelize
const { Op } = require('sequelize');

// 1. 查詢單條
const user = await User.findOne({ where: { name: 'Alice' } });

// 2. 查詢所有
const users = await User.findAll();

// 3. 按主鍵查詢
const user = await User.findByPk(10);

// 4. Where 條件
const users = await User.findAll({
  where: {
    name: { [Op.ne]: 'Alice' },
    age: { [Op.gt]: 18 },
    age: { [Op.between]: [20, 30] },
    name: { [Op.in]: ['Alice', 'Bob'] },
    name: { [Op.like]: '%li%' }
  }
});

// 5. Or 條件
const users = await User.findAll({
  where: {
    [Op.or]: [
      { name: 'Alice' },
      { name: 'Bob' }
    ]
  }
});

// 6. 選擇字段
const users = await User.findAll({
  attributes: ['name', 'age']
});

// 7. 排序
const users = await User.findAll({
  order: [['age', 'DESC'], ['name', 'ASC']]
});

// 8. 分頁
const users = await User.findAll({
  limit: 10,
  offset: 5
});

// 9. 計數
const count = await User.count({ where: { age: { [Op.gt]: 18 } } });
```

### 更新 (Update)

```go
package main

import (
    "gorm.io/gorm"
)

func updateExamples(db *gorm.DB) {
    // 1. 更新單個字段
    db.Model(&User{}).Where("id = ?", 1).Update("name", "Alice Smith")

    // 2. 更新多個字段（使用 struct）
    db.Model(&User{}).Where("id = ?", 1).Updates(User{
        Name: "Alice Smith",
        Age: 26,
    })

    // 3. 更新多個字段（使用 map，可以更新零值）
    db.Model(&User{}).Where("id = ?", 1).Updates(map[string]interface{}{
        "name": "Alice Smith",
        "age": 0,  // struct 方式會忽略零值，map 不會
    })

    // 4. 更新選定字段
    db.Model(&User{}).Where("id = ?", 1).
        Select("name", "age").
        Updates(User{Name: "Alice", Age: 26})

    // 5. 忽略某些字段
    db.Model(&User{}).Where("id = ?", 1).
        Omit("name").
        Updates(User{Name: "ignored", Age: 26})

    // 6. 批量更新
    db.Model(&User{}).Where("age < ?", 18).Update("active", false)

    // 7. 使用表達式更新
    db.Model(&User{}).Where("id = ?", 1).
        Update("age", gorm.Expr("age + ?", 1))

    // 8. 根據條件更新
    db.Model(&User{}).Where("active = ?", true).
        Updates(map[string]interface{}{
            "last_login": time.Now(),
        })

    // 9. 使用 Save（會更新所有字段，包括零值）
    var user User
    db.First(&user, 1)
    user.Name = "Alice"
    user.Age = 26
    db.Save(&user)

    // 10. 獲取受影響的行數
    result := db.Model(&User{}).Where("age > ?", 30).Update("active", false)
    fmt.Println("受影響的行數:", result.RowsAffected)
}
```

**Node.js 對比：**
```javascript
// Sequelize
// 1. 更新
await User.update(
  { name: 'Alice Smith', age: 26 },
  { where: { id: 1 } }
);

// 2. 更新包含零值
await User.update(
  { age: 0 },
  { where: { id: 1 } }
);

// 3. 增量更新
await User.increment('age', { by: 1, where: { id: 1 } });

// 4. 使用 save（需要先獲取實例）
const user = await User.findByPk(1);
user.name = 'Alice';
user.age = 26;
await user.save();

// 5. 獲取受影響的行數
const [affectedRows] = await User.update(
  { active: false },
  { where: { age: { [Op.gt]: 30 } } }
);
console.log('受影響的行數:', affectedRows);
```

### 刪除 (Delete)

```go
package main

import (
    "gorm.io/gorm"
)

func deleteExamples(db *gorm.DB) {
    // 1. 刪除單條記錄
    db.Delete(&User{}, 1)  // DELETE FROM users WHERE id = 1

    // 2. 根據條件刪除
    db.Where("name = ?", "Alice").Delete(&User{})

    // 3. 批量刪除
    db.Where("age < ?", 18).Delete(&User{})

    // 4. 刪除所有記錄（必須添加條件或使用 db.Session）
    // db.Delete(&User{})  // 不會執行，需要條件
    db.Where("1 = 1").Delete(&User{})  // 會執行
    db.Session(&gorm.Session{AllowGlobalUpdate: true}).Delete(&User{})

    // 5. 軟刪除（如果模型包含 gorm.DeletedAt）
    db.Delete(&User{}, 1)  // 實際執行：UPDATE users SET deleted_at = NOW() WHERE id = 1

    // 6. 查詢包含軟刪除的記錄
    db.Unscoped().Where("name = ?", "Alice").Find(&users)

    // 7. 永久刪除
    db.Unscoped().Delete(&User{}, 1)

    // 8. 獲取受影響的行數
    result := db.Where("age < ?", 18).Delete(&User{})
    fmt.Println("刪除的行數:", result.RowsAffected)
}
```

**Node.js 對比：**
```javascript
// Sequelize
// 1. 刪除
await User.destroy({ where: { id: 1 } });

// 2. 批量刪除
await User.destroy({ where: { age: { [Op.lt]: 18 } } });

// 3. 軟刪除（需要在模型中配置 paranoid: true）
await User.destroy({ where: { id: 1 } });  // 軟刪除

// 4. 查詢包含軟刪除的記錄
await User.findAll({ paranoid: false });

// 5. 永久刪除
await User.destroy({ where: { id: 1 }, force: true });

// 6. 刪除所有
await User.destroy({ where: {}, truncate: true });
```

## 3. 關聯關係

### Has One / Belongs To

```go
package main

import (
    "gorm.io/gorm"
)

// User has one Profile
type User struct {
    gorm.Model
    Name    string
    Profile Profile
}

type Profile struct {
    gorm.Model
    UserID uint   // 外鍵
    Bio    string
    Avatar string
}

// 自定義外鍵
type User2 struct {
    gorm.Model
    Name    string
    Profile Profile `gorm:"foreignKey:UID"`
}

type Profile2 struct {
    gorm.Model
    UID    uint   // 自定義外鍵名
    Bio    string
}

func hasOneExample(db *gorm.DB) {
    // 創建用戶和 Profile
    user := User{
        Name: "Alice",
        Profile: Profile{
            Bio: "Software Engineer",
            Avatar: "avatar.jpg",
        },
    }
    db.Create(&user)

    // 查詢並預加載 Profile
    var u User
    db.Preload("Profile").First(&u, user.ID)
    fmt.Println("User:", u.Name)
    fmt.Println("Bio:", u.Profile.Bio)

    // 創建關聯
    var user2 User
    db.First(&user2, 1)
    db.Model(&user2).Association("Profile").Append(&Profile{
        Bio: "New Bio",
    })
}
```

**Node.js 對比：**
```javascript
// Sequelize
const User = sequelize.define('User', { name: DataTypes.STRING });
const Profile = sequelize.define('Profile', {
  bio: DataTypes.TEXT,
  avatar: DataTypes.STRING
});

// 定義關聯
User.hasOne(Profile);
Profile.belongsTo(User);

// 創建
const user = await User.create({
  name: 'Alice',
  Profile: {
    bio: 'Software Engineer',
    avatar: 'avatar.jpg'
  }
}, {
  include: [Profile]
});

// 查詢並預加載
const user = await User.findByPk(1, {
  include: [Profile]
});
console.log('User:', user.name);
console.log('Bio:', user.Profile.bio);
```

### Has Many

```go
package main

import (
    "gorm.io/gorm"
)

// User has many Posts
type User struct {
    gorm.Model
    Name  string
    Posts []Post
}

type Post struct {
    gorm.Model
    UserID  uint
    Title   string
    Content string
}

func hasManyExample(db *gorm.DB) {
    // 創建用戶和多篇文章
    user := User{
        Name: "Alice",
        Posts: []Post{
            {Title: "Post 1", Content: "Content 1"},
            {Title: "Post 2", Content: "Content 2"},
        },
    }
    db.Create(&user)

    // 預加載文章
    var u User
    db.Preload("Posts").First(&u, user.ID)
    fmt.Printf("User %s has %d posts\n", u.Name, len(u.Posts))

    // 條件預加載
    db.Preload("Posts", "title LIKE ?", "%Post%").First(&u)

    // 添加關聯
    var user2 User
    db.First(&user2, 1)
    db.Model(&user2).Association("Posts").Append(&Post{
        Title: "New Post",
        Content: "New Content",
    })

    // 替換關聯
    db.Model(&user2).Association("Posts").Replace(&Post{
        Title: "Only Post",
    })

    // 刪除關聯
    var post Post
    db.First(&post)
    db.Model(&user2).Association("Posts").Delete(&post)

    // 清空關聯
    db.Model(&user2).Association("Posts").Clear()

    // 計數
    count := db.Model(&user2).Association("Posts").Count()
    fmt.Println("Post count:", count)
}
```

**Node.js 對比：**
```javascript
// Sequelize
User.hasMany(Post);
Post.belongsTo(User);

// 創建
const user = await User.create({
  name: 'Alice',
  Posts: [
    { title: 'Post 1', content: 'Content 1' },
    { title: 'Post 2', content: 'Content 2' }
  ]
}, {
  include: [Post]
});

// 預加載
const user = await User.findByPk(1, {
  include: [{
    model: Post,
    where: { title: { [Op.like]: '%Post%' } }
  }]
});

// 添加關聯
await user.addPost({ title: 'New Post', content: 'New Content' });

// 設置關聯
await user.setPosts([{ title: 'Only Post' }]);

// 移除關聯
await user.removePost(post);

// 計數
const count = await user.countPosts();
```

### Many To Many

```go
package main

import (
    "gorm.io/gorm"
)

// 多對多：Student - Course
type Student struct {
    gorm.Model
    Name    string
    Courses []Course `gorm:"many2many:student_courses;"`
}

type Course struct {
    gorm.Model
    Name     string
    Students []Student `gorm:"many2many:student_courses;"`
}

// 自定義連接表
type StudentCourse struct {
    StudentID uint
    CourseID  uint
    Grade     int
    CreatedAt time.Time
}

type Student2 struct {
    gorm.Model
    Name    string
    Courses []Course2 `gorm:"many2many:student_courses;"`
}

type Course2 struct {
    gorm.Model
    Name     string
    Students []Student2 `gorm:"many2many:student_courses;"`
}

func manyToManyExample(db *gorm.DB) {
    // 自動遷移（會創建連接表）
    db.AutoMigrate(&Student{}, &Course{})

    // 創建學生和課程
    student := Student{
        Name: "Alice",
        Courses: []Course{
            {Name: "Math"},
            {Name: "Physics"},
        },
    }
    db.Create(&student)

    // 預加載
    var s Student
    db.Preload("Courses").First(&s, student.ID)
    fmt.Printf("Student %s enrolled in %d courses\n",
        s.Name, len(s.Courses))

    // 添加關聯
    var course Course
    db.First(&course, "name = ?", "Chemistry")
    db.Model(&s).Association("Courses").Append(&course)

    // 查詢關聯
    var courses []Course
    db.Model(&s).Association("Courses").Find(&courses)

    // 刪除關聯
    db.Model(&s).Association("Courses").Delete(&course)

    // 使用自定義連接表
    db.SetupJoinTable(&Student2{}, "Courses", &StudentCourse{})
}
```

**Node.js 對比：**
```javascript
// Sequelize
const Student = sequelize.define('Student', { name: DataTypes.STRING });
const Course = sequelize.define('Course', { name: DataTypes.STRING });

// 定義多對多關聯
Student.belongsToMany(Course, { through: 'StudentCourse' });
Course.belongsToMany(Student, { through: 'StudentCourse' });

// 自定義連接表
const StudentCourse = sequelize.define('StudentCourse', {
  grade: DataTypes.INTEGER
});

Student.belongsToMany(Course, { through: StudentCourse });
Course.belongsToMany(Student, { through: StudentCourse });

// 創建
const student = await Student.create({
  name: 'Alice',
  Courses: [
    { name: 'Math' },
    { name: 'Physics' }
  ]
}, {
  include: [Course]
});

// 預加載
const student = await Student.findByPk(1, {
  include: [Course]
});

// 添加關聯
await student.addCourse(course);

// 移除關聯
await student.removeCourse(course);
```

## 4. 遷移與模式管理

```go
package main

import (
    "gorm.io/gorm"
)

func migrationExample(db *gorm.DB) {
    // 自動遷移（創建表）
    db.AutoMigrate(&User{}, &Post{}, &Profile{})

    // 檢查表是否存在
    hasTable := db.Migrator().HasTable(&User{})
    fmt.Println("Has users table:", hasTable)

    // 檢查列是否存在
    hasColumn := db.Migrator().HasColumn(&User{}, "Name")
    fmt.Println("Has name column:", hasColumn)

    // 添加列
    db.Migrator().AddColumn(&User{}, "Nickname")

    // 刪除列
    db.Migrator().DropColumn(&User{}, "Nickname")

    // 修改列
    db.Migrator().AlterColumn(&User{}, "Name")

    // 重命名列
    db.Migrator().RenameColumn(&User{}, "Name", "Username")

    // 創建索引
    db.Migrator().CreateIndex(&User{}, "Email")
    db.Migrator().CreateIndex(&User{}, "idx_user_email")

    // 刪除索引
    db.Migrator().DropIndex(&User{}, "Email")

    // 重命名表
    db.Migrator().RenameTable(&User{}, "new_users")

    // 刪除表
    db.Migrator().DropTable(&User{})
}
```

**Node.js 對比：**
```javascript
// Sequelize Migrations
module.exports = {
  up: async (queryInterface, Sequelize) => {
    // 創建表
    await queryInterface.createTable('Users', {
      id: {
        type: Sequelize.INTEGER,
        primaryKey: true,
        autoIncrement: true
      },
      name: Sequelize.STRING,
      email: Sequelize.STRING
    });

    // 添加列
    await queryInterface.addColumn('Users', 'nickname', Sequelize.STRING);

    // 刪除列
    await queryInterface.removeColumn('Users', 'nickname');

    // 修改列
    await queryInterface.changeColumn('Users', 'name', {
      type: Sequelize.STRING(100)
    });

    // 重命名列
    await queryInterface.renameColumn('Users', 'name', 'username');

    // 添加索引
    await queryInterface.addIndex('Users', ['email']);

    // 重命名表
    await queryInterface.renameTable('Users', 'new_users');
  },

  down: async (queryInterface) => {
    await queryInterface.dropTable('Users');
  }
};
```

## 5. 鉤子 (Hooks)

```go
package main

import (
    "gorm.io/gorm"
    "golang.org/x/crypto/bcrypt"
)

type User struct {
    gorm.Model
    Username string
    Password string
    Email    string
}

// BeforeCreate - 創建前
func (u *User) BeforeCreate(tx *gorm.DB) error {
    // 加密密碼
    if u.Password != "" {
        hash, err := bcrypt.GenerateFromPassword(
            []byte(u.Password), bcrypt.DefaultCost)
        if err != nil {
            return err
        }
        u.Password = string(hash)
    }
    return nil
}

// AfterCreate - 創建後
func (u *User) AfterCreate(tx *gorm.DB) error {
    fmt.Println("User created:", u.ID)
    // 發送歡迎郵件等
    return nil
}

// BeforeUpdate - 更新前
func (u *User) BeforeUpdate(tx *gorm.DB) error {
    // 如果密碼被修改，重新加密
    if tx.Statement.Changed("Password") {
        hash, err := bcrypt.GenerateFromPassword(
            []byte(u.Password), bcrypt.DefaultCost)
        if err != nil {
            return err
        }
        u.Password = string(hash)
    }
    return nil
}

// AfterFind - 查詢後
func (u *User) AfterFind(tx *gorm.DB) error {
    // 清空敏感信息
    u.Password = ""
    return nil
}

// BeforeDelete - 刪除前
func (u *User) BeforeDelete(tx *gorm.DB) error {
    // 級聯刪除相關數據
    return nil
}
```

**Node.js 對比：**
```javascript
// Sequelize Hooks
const User = sequelize.define('User', {
  username: DataTypes.STRING,
  password: DataTypes.STRING,
  email: DataTypes.STRING
}, {
  hooks: {
    // 創建前
    beforeCreate: async (user) => {
      if (user.password) {
        user.password = await bcrypt.hash(user.password, 10);
      }
    },

    // 創建後
    afterCreate: async (user) => {
      console.log('User created:', user.id);
      // 發送歡迎郵件
    },

    // 更新前
    beforeUpdate: async (user) => {
      if (user.changed('password')) {
        user.password = await bcrypt.hash(user.password, 10);
      }
    },

    // 查詢後
    afterFind: (users) => {
      if (Array.isArray(users)) {
        users.forEach(user => user.password = undefined);
      } else if (users) {
        users.password = undefined;
      }
    },

    // 刪除前
    beforeDestroy: async (user) => {
      // 級聯刪除
    }
  }
});
```

## 完整示例：博客系統

```go
package main

import (
    "fmt"
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

// 模型定義
type User struct {
    gorm.Model
    Username string `gorm:"uniqueIndex;size:50;not null"`
    Email    string `gorm:"uniqueIndex;size:100;not null"`
    Password string `gorm:"size:255;not null"`
    Posts    []Post
}

type Post struct {
    gorm.Model
    UserID    uint
    User      User
    Title     string `gorm:"size:200;not null"`
    Content   string `gorm:"type:text"`
    Published bool   `gorm:"default:false"`
    Tags      []Tag  `gorm:"many2many:post_tags;"`
}

type Tag struct {
    gorm.Model
    Name  string `gorm:"uniqueIndex;size:50"`
    Posts []Post `gorm:"many2many:post_tags;"`
}

// Repository
type BlogRepository struct {
    db *gorm.DB
}

func NewBlogRepository(db *gorm.DB) *BlogRepository {
    return &BlogRepository{db: db}
}

// User 操作
func (r *BlogRepository) CreateUser(username, email, password string) (*User, error) {
    user := &User{
        Username: username,
        Email:    email,
        Password: password,
    }
    result := r.db.Create(user)
    return user, result.Error
}

func (r *BlogRepository) GetUserByID(id uint) (*User, error) {
    var user User
    result := r.db.Preload("Posts").First(&user, id)
    return &user, result.Error
}

// Post 操作
func (r *BlogRepository) CreatePost(userID uint, title, content string, tags []string) (*Post, error) {
    var tagModels []Tag
    for _, tagName := range tags {
        var tag Tag
        r.db.FirstOrCreate(&tag, Tag{Name: tagName})
        tagModels = append(tagModels, tag)
    }

    post := &Post{
        UserID:  userID,
        Title:   title,
        Content: content,
        Tags:    tagModels,
    }

    result := r.db.Create(post)
    return post, result.Error
}

func (r *BlogRepository) GetPublishedPosts(page, pageSize int) ([]Post, int64, error) {
    var posts []Post
    var total int64

    query := r.db.Model(&Post{}).Where("published = ?", true)

    query.Count(&total)

    result := query.
        Preload("User").
        Preload("Tags").
        Order("created_at DESC").
        Limit(pageSize).
        Offset((page - 1) * pageSize).
        Find(&posts)

    return posts, total, result.Error
}

func (r *BlogRepository) PublishPost(postID uint) error {
    return r.db.Model(&Post{}).Where("id = ?", postID).
        Update("published", true).Error
}

func (r *BlogRepository) GetPostsByTag(tagName string) ([]Post, error) {
    var tag Tag
    err := r.db.Preload("Posts.User").
        Where("name = ?", tagName).
        First(&tag).Error

    if err != nil {
        return nil, err
    }

    return tag.Posts, nil
}

// 統計
func (r *BlogRepository) GetUserStats(userID uint) (map[string]interface{}, error) {
    stats := make(map[string]interface{})

    // 總文章數
    var totalPosts int64
    r.db.Model(&Post{}).Where("user_id = ?", userID).Count(&totalPosts)
    stats["total_posts"] = totalPosts

    // 已發布文章數
    var publishedPosts int64
    r.db.Model(&Post{}).Where("user_id = ? AND published = ?", userID, true).
        Count(&publishedPosts)
    stats["published_posts"] = publishedPosts

    // 最近發布日期
    var lastPost Post
    r.db.Where("user_id = ? AND published = ?", userID, true).
        Order("created_at DESC").
        First(&lastPost)
    stats["last_post_date"] = lastPost.CreatedAt

    return stats, nil
}

func main() {
    // 連接數據庫
    dsn := "root:password@tcp(127.0.0.1:3306)/blog?parseTime=true"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        panic("連接數據庫失敗")
    }

    // 自動遷移
    db.AutoMigrate(&User{}, &Post{}, &Tag{})

    // 創建 Repository
    repo := NewBlogRepository(db)

    // 創建用戶
    user, _ := repo.CreateUser("alice", "alice@example.com", "password123")
    fmt.Printf("創建用戶: %s (ID: %d)\n", user.Username, user.ID)

    // 創建文章
    post, _ := repo.CreatePost(user.ID, "GORM 入門", "GORM 是一個很棒的 ORM...",
        []string{"Go", "Database", "GORM"})
    fmt.Printf("創建文章: %s (ID: %d)\n", post.Title, post.ID)

    // 發布文章
    repo.PublishPost(post.ID)

    // 獲取已發布文章
    posts, total, _ := repo.GetPublishedPosts(1, 10)
    fmt.Printf("已發布文章: %d 篇\n", total)
    for _, p := range posts {
        fmt.Printf("- %s by %s\n", p.Title, p.User.Username)
    }

    // 獲取用戶統計
    stats, _ := repo.GetUserStats(user.ID)
    fmt.Printf("用戶統計: %+v\n", stats)
}
```

## 重點總結

### GORM vs Sequelize

| 特性 | GORM | Sequelize |
|------|------|-----------|
| **模型定義** | Struct + Tags | Object + Types |
| **遷移** | AutoMigrate | Migration files |
| **查詢** | 鏈式方法 | 鏈式方法 |
| **關聯** | Preload | include |
| **鉤子** | 方法定義 | hooks 配置 |
| **事務** | Begin/Commit | transaction |
| **軟刪除** | DeletedAt | paranoid |

### 最佳實踐

1. **總是使用 Preload 避免 N+1 查詢**
2. **使用事務保證數據一致性**
3. **合理使用索引提升查詢性能**
4. **使用 Context 控制超時**
5. **檢查 Error 和 RowsAffected**

## 練習題

1. **基礎**：實現一個簡單的電商商品管理系統（Product CRUD）
2. **中級**：實現一個帶標籤和分類的文章系統
3. **高級**：實現一個多租戶博客系統，支持軟刪除和數據隔離
4. **實戰**：實現一個完整的訂單系統，包含用戶、商品、訂單、訂單項的關聯關係

## 下一步

下一章將學習 PostgreSQL 的 Go 集成，包括特定功能如 JSON、數組等高級特性。
