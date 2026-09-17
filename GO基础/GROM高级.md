# GROM高级

## 一、链式查询

GROM是链式API，可以不断拼接条件，最后用Find/First/Scan执行SQL.

**关键点**：链式方法不会立即发SQL,只有调用Find/First/Count这种终结方法，才会执行数据库查询

**常用条件示例**

```go
var users []User
// Where 多条件；Order排序；Limit限制行数；Offset偏移
err := db.
    Where("age > ?", 18).
    Where("name LIKE ?", "%Alice%").
    Order("id DESC").
    Limit(10).
    Offset(0).
    Find(&users).Error
```

生成**SQL**

```sql
SELECT * FROM users WHERE age>18 AND name LIKE '%Alice%' ORDER BY id DESC LIMIT 10 OFFSET 0;
```

#### 1. Where 多种写法

```go
// 方式1：字符串+占位符（推荐，防注入）
db.Where("age > ? AND name = ?", 18, "Alice")

// 方式2：map条件（等于匹配）
db.Where(map[string]any{"age":18, "name":"Alice"})

// 方式3：结构体（只会用非零值做等于匹配）
db.Where(&User{Name:"Alice", Age:18})
```

#### 2. OR 条件

```go
// age>18 OR name = Bob
db.Where("age>?",18).Or("name=?","Bob").Find(&users)
```

#### 3.Select 指定查询字段（不要总是 select *）

```go
// 只查询 id,name,age，减少传输开销
db.Select("id","name","age").Find(&users)
```

#### 4.Order 排序

```go
db.Order("id desc") // id降序
db.Order("age asc, id desc") // 多字段排序
```

#### 5.Limit & Offset 分页

- Limit：最多返回多少条（页大小 pageSize）
- Offset：跳过前面多少条 `offset = (pageNum - 1)*pageSize`

```go
pageNum := 1
pageSize := 10
offset := (pageNum -1)*pageSize
var users []User
db.Limit(pageSize).Offset(offset).Find(&users)
```

#### 6. Count 统计总数（分页需要返回总条数）

```go
var total int64
db.Where("age>?",18).Count(&total)
```

> 分页接口标准逻辑：查列表 + 查总条数，前端用来渲染分页组件。

# 二、链式调用原理

```go
db.Where("age>?",18).Order("id desc").Find(&users)
```

1. `db` 是 `*gorm.DB` 对象
2. `Where()` 返回**新的`\*gorm.DB`对象**，不是修改原对象！
3. 每次链式方法，只是**把条件保存到对象里，不访问数据库**
4. 直到调用 `Find/First/Count/Update/Delete`，才会把所有条件拼成 SQL，访问数据库。

> 重要坑：链式对象是**拷贝**，复用变量会踩坑！

```go
// 错误示范
query := db.Where("age>?",18)
var u1 []User
query.Limit(10).Find(&u1) // 这里修改query内部条件
var u2 []User
query.Find(&u2) // 第二次查询，还带上Limit(10)，不是预期结果！
```

原因：query 是同一个对象，Limit 修改了内部条件。 ✅ 规范：**每次查询重新链式构造，不要复用 query 变量**。

## 三、子查询 & Exists

```go
// 子查询：查询有订单的用户
var users []User
db.Where("id IN (SELECT DISTINCT user_id FROM orders)").Find(&users)

// Exists 判断：是否存在符合条件记录
var exists bool
db.Select("1").Where("name=?","Alice").Take(&exists)
```

## 四、事务进阶

#### 1. 基础回顾

```go
tx := db.Begin()
// 操作失败就 Rollback，成功 Commit
if tx.Create(&user).Error != nil {
    tx.Rollback()
    return
}
tx.Commit()
```

#### 2. 保存点 SavePoint（事务内局部回滚，不回滚整个事务）

大事务中，某一段失败只回滚这一段，前面的保留。

```go
tx := db.Begin()
tx.Create(&user1)
tx.SavePoint("sp1") // 设置保存点

tx.Create(&user2)
if err != nil {
    tx.RollbackTo("sp1") // 回滚到sp1，user1保留，user2撤销
}
tx.Commit()
```

#### 3. 嵌套事务（GORM 本质是保存点实现）

GORM 的嵌套事务不是数据库原生嵌套，底层是 SavePoint。

```go
tx := gormDB.Begin()
tx.Create(&User{Name:"A"})

// 内层事务，底层自动创建savepoint
tx.Begin()
tx.Create(&User{Name:"B"})
tx.Rollback() // 只会撤销B，A还在事务中

tx.Commit()
```

## 五、实战：Gin + GORM 分页用户列表接口

```go
// GetUserList 分页查询用户列表
func GetUserList(c *gin.Context) {
    type Req struct {
        Page     int    `form:"page" binding:"min=1"`
        PageSize int    `form:"page_size" binding:"min=1,max=100"`
        Name     string `form:"name"`
    }
    var req Req
    if err := c.ShouldBindQuery(&req); err != nil {
        c.JSON(400, gin.H{"msg":"参数错误"})
        return
    }

    query := gormDB.Model(&User{})
    if req.Name != "" {
        query = query.Where("name LIKE ?", "%"+req.Name+"%")
    }

    // 查询总数
    var total int64
    query.Count(&total)

    // 查询列表
    var users []User
    offset := (req.Page - 1) * req.PageSize
    query.Order("id DESC").Limit(req.PageSize).Offset(offset).Find(&users)

    c.JSON(200, gin.H{
        "data": users,
        "total": total,
        "page": req.Page,
        "page_size": req.PageSize,
    })
}
```

**路由：**

```go
r.GET("/user/list", GetUserList)
```