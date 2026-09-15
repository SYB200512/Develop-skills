# GORM与sqlx

## 1.GROM基础

GROM把Go的结构体和数据库映射起来，不用手写大量基础SQL,用Go代码操作数据库

### 1.1结构体

```go
type User struct {
    ID        uint      `gorm:"primaryKey"`       // 主键
    Name      string    `gorm:"size:100;not null"`// 字段长度100，非空
    Email     string    `gorm:"uniqueIndex;size:255"` // 唯一索引
    Age       int       `gorm:"default:18"`       // 默认值18
    CreatedAt time.Time                            // GORM自动维护创建时间
    UpdatedAt time.Time                            // GORM自动维护更新时间
}
```

GORM 约定：

- 结构体名 `User` 默认对应表名 `users`（小写复数）
- `CreatedAt/UpdatedAt` 是内置时间戳字段，Create/Update 时自动填充
- tag 可以控制：主键、索引、长度、默认值、是否为空

### 1.2CRUD基础操作

```go
package main

import (
	"fmt"
	"time"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

// User GORM模型
type User struct {
	ID        uint      `gorm:"primaryKey"`          // 主键，自增
	Name      string    `gorm:"size:100;not null"`   // 字段长度，非空
	Email     string    `gorm:"uniqueIndex;size:255"`// 唯一索引
	Age       int       `gorm:"default:18"`          // 默认值
	CreatedAt time.Time
	UpdatedAt time.Time
	// 开启软删除需要加上下面这行
	DeletedAt gorm.DeletedAt `gorm:"index"`
}

func main() {
	// 1. 数据库连接DSN，修改成你自己数据库账号密码库名
	dsn := "root:123456@tcp(127.0.0.1:3306)/testdb?charset=utf8mb4&parseTime=True&loc=Local"
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		// 开启日志，打印SQL语句
		Logger: gorm.DefaultLogger.LogMode(true),
	})
	if err != nil {
		panic("数据库连接失败:" + err.Error())
	}

	// 自动迁移表结构（开发环境使用）
	err = db.AutoMigrate(&User{})
	if err != nil {
		panic("迁移失败:" + err.Error())
	}
	fmt.Println("表迁移完成")

	// ====================== 【Create 创建】 ======================
	user := User{Name: "Alice", Email: "alice@test.com", Age: 20}
	// 传入指针，插入成功后自动回填主键user.ID
	err = db.Create(&user).Error
	if err != nil {
		panic("创建用户失败:" + err.Error())
	}
	fmt.Printf("创建成功，用户ID=%d\n", user.ID)

	// ====================== 【Read 查询】 ======================
	var u User
	// First：按主键查询id=1，找不到返回 gorm.ErrRecordNotFound
	err = db.First(&u, 1).Error
	if err != nil {
		panic("按主键查询失败:" + err.Error())
	}
	fmt.Printf("按主键查询结果：name=%s, email=%s\n", u.Name, u.Email)

	// 条件查询，where 条件 + First 查第一条
	err = db.Where("name = ?", "Alice").First(&u).Error
	if err != nil {
		panic("条件查询失败:" + err.Error())
	}
	fmt.Printf("条件查询结果：name=%s\n", u.Name)

	// 多条查询，Find 放到切片
	var users []User
	err = db.Where("age > ?", 18).Find(&users).Error
	if err != nil {
		panic("批量查询失败:" + err.Error())
	}
	fmt.Println("age>18 的用户列表：")
	for _, item := range users {
		fmt.Printf("id:%d name:%s age:%d\n", item.ID, item.Name, item.Age)
	}

	// ====================== 【Update 更新】 ======================
	// 更新单个字段
	err = db.Model(&u).Update("name", "Bob").Error
	if err != nil {
		panic("更新单个字段失败:" + err.Error())
	}
	fmt.Println("单个字段更新完成")

	// 更新多个字段，Updates接收结构体，只更新非零值
	err = db.Model(&u).Updates(User{Name: "Bob", Age: 22}).Error
	if err != nil {
		panic("多字段更新失败:" + err.Error())
	}
	fmt.Println("多字段更新完成")

	// ⚠️高危坑点演示：
	// db.Updates(User{Name:"xxx"}) 不写Model，不带条件，会更新全表！禁止这么写！
	// db.Updates(User{Name:"danger"})

	// ====================== 【Delete 删除】 ======================
	// 软删除：更新 deleted_at 字段，不会真实DELETE
	err = db.Delete(&u).Error
	if err != nil {
		panic("软删除失败:" + err.Error())
	}
	fmt.Println("软删除执行完成")

	// 软删除后，正常查询会自动忽略这条数据
	var checkUser User
	err = db.First(&checkUser, u.ID).Error
	fmt.Printf("软删除后查询是否找到：%v\n", err)

	// Unscoped() 永久删除，执行真实DELETE SQL
	err = db.Unscoped().Delete(&u).Error
	if err != nil {
		panic("永久删除失败:" + err.Error())
	}
	fmt.Println("永久删除执行完成")
}

```

GORM 默认**软删除**，表会自动增加 `DeletedAt` 字段，Delete 操作只是写删除时间，查询时自动过滤已删除数据。

### 1.3关联查询

业务场景：**一个用户 (User) 有多条订单 (Order)**，一对多关系

```go
type Order struct {
	ID        uint
	UserID    uint    // 外键，关联 user.ID
	Amount    float64 // 订单金额
	OrderNo   string  // 订单编号
	CreatedAt time.Time
}

type User struct {
	ID        uint
	Name      string
	Email     string
	Age       int
	Orders    []Order // 一对多关联：一个用户对应多个订单
	CreatedAt time.Time
	UpdatedAt time.Time
}
```

####  N+1 错误写法（新手高频踩坑）

```go
var user User
db.First(&user, 1) // 第1条SQL：查询用户 (1次)
// 循环取出用户订单，每循环一次执行一次SQL查询订单 N次
// 总共 1+N 条SQL，数据量大的时候性能爆炸！
```

####  Preload 预加载（解决 N+1）

```go
var user User
// Preload("Orders")：JOIN逻辑，一次性把用户+对应的订单全部查出来
err := db.Preload("Orders").First(&user, 1).Error
```

> Preload：提前把关联数据一次性加载，只产生 2 条 SQL，避免循环查询。 适用：一对一、一对多、多对多关联场景。

### 1.4 AutoMigrate

```go
err := db.AutoMigrate(&User{}, &Order{})
```

作用：根据结构体自动创建表、新增字段、新增索引。  限制：**不会删除数据库原有字段，不会修改字段类型**。 ✅ 适合开发环境快速建表；❌ 生产环境谨慎使用，生产建议使用迁移脚本管理表结构变更。

### 1.5GROM事务

```go
tx := db.Begin() // 开启事务
if tx.Error != nil {
    panic(tx.Error)
}

// 操作1
if err := tx.Create(&user).Error; err != nil {
    tx.Rollback() // 出错回滚
    return err
}
// 操作2
if err := tx.Create(&order).Error; err != nil {
    tx.Rollback()
    return err
}

tx.Commit() // 提交事务
```

## 二、sqlx详解

sqlx **不是 ORM**，它只是标准库 `database/sql` 的增强封装。
不会自动生成 SQL，SQL 全部手写；核心能力：**直接把查询结果映射到结构体**，省去原生 sql 繁琐的 row.Scan ()。

原生 database/sql 需要手动定义变量 scan，非常繁琐；sqlx 简化了映射。

## sqlx 基础 API

- `Get(dest interface{}, query string, args ...interface{}) error`：查询**单行**，映射到结构体
- `Select(dest interface{}, query string, args ...interface{}) error`：查询**多行**，映射到结构体切片

```go
import "github.com/jmoiron/sqlx"

type User struct {
	ID    uint
	Name  string
	Email string
	Age   int
}

var u User
// 查询单行
err := db.Get(&u, `SELECT id,name,email,age FROM users WHERE id = $1`, 1)

// 查询多行
var users []User
err := db.Select(&users, `SELECT id,name,email,age FROM users WHERE age > $1`, 18)
```

优势：

1. **完全掌控 SQL**：复杂多表 JOIN、子查询、聚合统计，SQL 自己手写，可以直接拿给 DBA 做 explain 调优，没有 ORM 生成 SQL 黑盒问题。
2. **性能更好**：没有 GORM 大量反射、链式构造 SQL 的开销；大批量数据场景优势明显。
3. 支持命名参数（`:name`），长 SQL 可读性更好。

| 场景                                  | 推荐 | 理由                                                         |
| ------------------------------------- | ---- | ------------------------------------------------------------ |
| 简单 CRUD，单表增删改查               | GORM | 开发效率高，代码简洁，不用手写 SQL；自带时间戳、软删除、事务 |
| 复杂查询、多表 JOIN、子查询、报表统计 | sqlx | SQL 直观可控，GORM 写复杂关联可读性差，生成 SQL 难以优化     |
| 批量插入、大批量数据操作              | sqlx | 性能更好，减少反射开销                                       |
| 数据库表结构迁移                      | GORM | AutoMigrate 快速维护表结构，开发环境方便                     |

## 三、完整案例

业务场景：用户和订单系统 需求：

1. 创建用户（GORM Create）
2. 创建用户订单（GORM Create，事务保证用户订单同时创建）
3. 查询用户 + 关联订单（GORM Preload）
4. 统计：查询所有用户订单总金额，多表 JOIN 聚合（sqlx）

```go
package main

import (
	"fmt"
	"time"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"github.com/jmoiron/sqlx"
)

// GORM模型定义
type User struct {
	ID        uint      `gorm:"primaryKey"`
	Name      string    `gorm:"size:100;not null"`
	Email     string    `gorm:"uniqueIndex;size:255"`
	Age       int       `gorm:"default:18"`
	CreatedAt time.Time
	UpdatedAt time.Time
	Orders    []Order //一对多关联
}

type Order struct {
	ID        uint
	UserID    uint
	Amount    float64
	OrderNo   string
	CreatedAt time.Time
}

// 报表结构体，用于sqlx接收JOIN统计结果
type UserOrderStat struct {
	UserName string  `db:"user_name"`
	OrderSum float64 `db:"sum_amount"`
}

func main() {
	// ========== 1. GORM 连接数据库 ==========
	dsn := "root:123456@tcp(127.0.0.1:3306)/testdb?charset=utf8mb4&parseTime=True&loc=Local"
	gormDB, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		panic(err)
	}
	// 自动建表（开发环境）
	gormDB.AutoMigrate(&User{}, &Order{})

	// ========== 2. GORM：事务创建用户和订单 ==========
	tx := gormDB.Begin()
	user := User{Name: "Alice", Email: "alice@test.com", Age: 20}
	if err := tx.Create(&user).Error; err != nil {
		tx.Rollback()
		panic(err)
	}
	order := Order{UserID: user.ID, Amount: 99.9, OrderNo: "ORD20260526"}
	if err := tx.Create(&order).Error; err != nil {
		tx.Rollback()
		panic(err)
	}
	tx.Commit()
	fmt.Printf("创建成功，用户ID:%d\n", user.ID)

	// ========== 3. GORM Preload 查询用户+订单（关联查询） ==========
	var findUser User
	err = gormDB.Preload("Orders").First(&findUser, user.ID).Error
	if err != nil {
		panic(err)
	}
	fmt.Printf("用户姓名：%s，订单金额：%.2f\n", findUser.Name, findUser.Orders[0].Amount)

	// ========== 4. sqlx 复杂JOIN统计：查询每个用户订单总金额 ==========
	// sqlx底层可以复用同一个数据库连接
	sqlDB, err := gormDB.DB()
	if err != nil {
		panic(err)
	}
	sqlxDB := sqlx.NewDb(sqlDB, "mysql")

	var stats []UserOrderStat
	// 手写JOIN聚合SQL
	sql := `
		SELECT u.name AS user_name, SUM(o.amount) AS sum_amount
		FROM users u
		LEFT JOIN orders o ON u.id = o.user_id
		GROUP BY u.name
	`
	err = sqlxDB.Select(&stats, sql)
	if err != nil {
		panic(err)
	}
	fmt.Println("==== 用户订单统计报表(sqlx) ====")
	for _, s := range stats {
		fmt.Printf("用户：%s，订单总金额：%.2f\n", s.UserName, s.OrderSum)
	}
}
```

#### 案例知识点串联总结

1. **单表 CRUD、事务、关联查询**：使用 GORM，减少手写 SQL，开发效率高；Preload 解决 N+1 问题。
2. **多表 JOIN 聚合报表**：使用 sqlx，手写 SQL，方便 SQL 调优，适合复杂统计场景。
3. 同一个数据库连接，可以同时给 GORM 和 sqlx 共用，项目中混合使用。

# 四、常见坑汇总

## GORM 坑

1. `db.Model(&u).Update` 忘记 Model，会更新全表！
2. Preload 只加载指定关联，多层关联需要嵌套 Preload
3. 软删除：查询默认忽略 DeletedAt，要用 Unscoped 才能查到已删除记录
4. 结构体零值：Updates 更新时，int=0、string="" 这类零值不会更新，如果需要更新零值要使用 map

## sqlx 坑

1. MySQL 占位符 `?`，PostgreSQL 占位符 `$1`，不要混用
2. 结构体字段 tag 必须写 `db:"column_name"`，否则 sqlx 无法映射查询结果
3. Select/Get 传入必须是指针，否则无法回填数据