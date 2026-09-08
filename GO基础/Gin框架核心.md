# Gin框架核心

## 1.路由定义

```go
func main(){
    r := gin.default()//创建gin引擎实例
    
    //基本路由
    r.GET("/hello",func(c *gin.Context){
        c.JSON(http.StatusOK,gin.H{
            "message":"hello gin"
        })
    })
    
    r.POST("/user",func(c *gin.Context){
        c.JSON(200,gin.H{
            "msg":创建用户
        })
    })
    
    //路径参数：变量写在URL路径里，格式：/：变量名
    //param()返回的永远是字符串，数字需要自己转strconv.Atoi()
    r.GET("/user/:id"，func(c *gin.Context){
        //获取路径的参数
        id := c.Param("id")
        c.JSON(200,gin.H{
            "user_id":id,
        })
    })
    
    
    //查询参数
    //url格式：/list?page=1&size=2&name=zhangsan
    r.GET("/list", func(c *gin.Context) {
    // 1. GetQuery：拿参数，不存在返回空字符串
    page := c.Query("page")

    // 2. GetQuery的变种：DefaultQuery，参数不存在给默认值
    size := c.DefaultQuery("size", "20")

    // 3. GetQueryArray 获取数组参数：?ids=1&ids=2&ids=3
    ids := c.QueryArray("ids")

    c.JSON(200, gin.H{
        "page": page,
        "size": size,
        "ids": ids,
    })
    })
    
    
    
    //路由分组
    // admin路由组，统一前缀 /admin
adminGroup := r.Group("/admin"){
    adminGroup.GET("/user", func(c *gin.Context) {
        c.JSON(200, gin.H{"msg":"管理员用户列表"})
    })
    adminGroup.GET("/category", func(c *gin.Context) {
        c.JSON(200, gin.H{"msg":"分类管理"})
    })
}
    
}
```

### 其他知识点补充

#### 1. Handler 可以传多个函数（中间件链）

```go
// 先执行Middleware1，再执行业务handler
r.GET("/test", Middleware1, func(c *gin.Context){})
```

`c.Next()` 调用下一个 handler；`c.Abort()` 终止后续 handler 执行（鉴权失败直接返回）。

#### 2. 404 路由

没有任何路由匹配时触发

```go
r.NoRoute(func(c *gin.Context) {
    c.JSON(404, gin.H{"code":404,"msg":"接口不存在"})
})
```

## 2.中间件

一条路由可以挂载**多个 Handler 函数**，按顺序依次执行，中间件就是放在业务 handler**之前 / 之后**执行的函数，可以做：日志、鉴权、跨域、参数校验、panic 捕获。

Gin 请求处理链路：`请求进来 → 中间件1 → 中间件2 → 业务handler → 返回响应`

###### 两个核心函数（必须搞懂）

1. **`c.Next()`**

> 执行**后面剩下的 Handler 链**。调用`c.Next()`之后，会去执行后续中间件 / 业务 handler；**等后面全部执行完毕，代码会回到 `c.Next()` 的下一行继续执行**。 可以实现：请求前逻辑（c.Next 之前）、请求后逻辑（c.Next 之后）

1. **`c.Abort()` / `c.AbortWithStatusJSON()`**

> **终止整条 Handler 链条，不再往下执行后续中间件和业务处理函数，直接返回响应**。鉴权失败就用这个，直接拦截请求。

###### 1.全局中间件

```go
// 全局中间件，所有请求，全部都会走这两个中间件
r.Use(gin.Logger())
r.Use(gin.Recovery())

```

- `r.Use(中间件)`：**注册全局中间件，项目里每一条 HTTP 请求，都会执行**。

1. `gin.Logger()`：Gin 内置，**日志中间件**。打印请求日志：请求方法、url、状态码、耗时。
2. `gin.Recovery()`：Gin 内置，**崩溃恢复中间件**。


如果 handler 代码发生 panic，程序不会直接崩溃退出；捕获 panic，返回 500 错误给前端。**生产环境必开**。

###### 2.自定义鉴权中间件

```go
// 返回类型 gin.HandlerFunc，这就是中间件类型
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // ========== 请求前：鉴权逻辑 ==========
        token := c.GetHeader("Authorization")
        if token == "" {
            // 没有token，拦截！终止后续所有处理，直接返回JSON
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "missing authorization header",
            })
            return
        }

        // token校验通过，把user_id存到上下文，业务接口可以拿
        c.Set("user_id", "123")

        // ✅放行！执行后面的中间件/业务handler
        c.Next()

        // ========== c.Next()之后：请求完成之后才执行的代码 ==========
        // 这里可以写请求结束后的逻辑，比如统计耗时
    }
}

```

###### 3.路由组级别中间件

```go
authorized := r.Group("/admin")
authorized.Use(AuthMiddleware()) // ✅路由组中间件
{
    authorized.GET("/dashboard", adminDashboard)
}

```

- authorized.Use(AuthMiddleware())`：**这个组下面所有路由，全部自动执行 AuthMiddleware 鉴权**。
- `/admin/dashboard` 请求：进来先走鉴权中间件 →鉴权成功才执行`adminDashboard`业务函数。
- 其他不在这个 Group 的路由，**不受影响**，不会执行鉴权。

###### 4.单路由级别中间件

```go
r.GET("/protected", AuthMiddleware(), protectedHandler)

```

！！只给这一条路由生效，其他路由不受影响



##### 简单案例助理解

```
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"net/http"
)

// 自定义全局中间件1
func GlobalMid1() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Println("【全局中间件1】请求进来")
		c.Next()
		fmt.Println("【全局中间件1】请求结束")
	}
}

// 自定义鉴权中间件：路由组使用
func AuthMid() gin.HandlerFunc {
	return func(c *gin.Context) {
		fmt.Println("【鉴权中间件】开始校验token")
		token := c.GetHeader("token")
		if token == "" {
			// 没有token直接拦截
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"msg":"未登录"})
			return
		}
		// 存入上下文
		c.Set("username", "testUser")
		c.Next()
		fmt.Println("【鉴权中间件】校验完成")
	}
}

func main() {
	r := gin.Default()
	// 注册全局中间件：所有请求都会执行
	r.Use(GlobalMid1())

	// 基础路由 GET
	r.GET("/hello", func(c *gin.Context) {
		fmt.Println("【业务handler】执行hello接口")
		c.JSON(http.StatusOK, gin.H{"msg":"hello 普通接口，不需要登录"})
	})

	// 路由分组，挂载鉴权中间件，组内全部接口需要token
	adminGroup := r.Group("/admin")
	adminGroup.Use(AuthMid())
	{
		adminGroup.GET("/info", func(c *gin.Context) {
			fmt.Println("【业务handler】执行admin/info接口")
			// 从上下文取出中间件设置的值
			name, _ := c.Get("username")
			c.JSON(http.StatusOK, gin.H{"msg":"后台接口","username":name})
		})
	}

	// 单路由级别中间件：只给这一条路由生效
	r.GET("/safe", AuthMid(), func(c *gin.Context) {
		fmt.Println("【业务handler】执行safe接口")
		c.JSON(http.StatusOK, gin.H{"msg":"单路由保护接口"})
	})

	r.Run(":8080")
}

```

测试一：访问普通接口：GET http://127.0.0.1:8080/hello

控制台

```
【全局中间件1】请求进来
【业务handler】执行hello接口
【全局中间件1】请求结束
```

测试二：访问 admin 接口，请求头不带 token：GET http://127.0.0.1:8080/admin/info`，header 不加`token

```
【全局中间件1】请求进来
【鉴权中间件】开始校验token
【全局中间件1】请求结束
```

测试三：访问 admin 接口，请求头带上 token: abc

```
【全局中间件1】请求进来
【鉴权中间件】开始校验token
【业务handler】执行admin/info接口
【鉴权中间件】校验完成
【全局中间件1】请求结束
```

## 3.请求绑定与验证

请求绑定（ShouldBindJSON）：把前端 HTTP 请求里的数据，解析赋值到 Go 的结构体变量；
验证（binding tag）：绑定的同时，按照结构体 `binding` 标签规则自动校验数据是否合法

### 1.结构体定义

```go
type CreateUserRequest struct {
    Name     string `json:"name" binding:"required,min=2,max=50"`
    Email    string `json:"email" binding:"required,email"`
    Age      int    `json:"age" binding:"gte=0,lte=150"`
    Password string `json:"password" binding:"required,min=8"`
}
```

1. `json:"name"`

- 作用：**JSON 字段映射**。前端 JSON key 是`name`，映射到 Go 结构体字段`Name`。

>
> Go 结构体字段大写，JSON 字段可以小写，靠这个 tag 对应。

2. `binding:"required,min=2,max=50"`

>
> 校验规则，多个规则逗号分隔。

`required`字段**必填**，不能不传、不能为空字符串

`min=2`字符串最小长度；数字则代表最小值

`max=50`字符串最大长度；数字代表最大值

`email`校验邮箱格式`gte=0`大于等于（greater or equal）

`lte=150`小于等于（less or equal

### 2.`c.ShouldBindJSON(&req)` 请求绑定

```go
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{
        "error": validationError(err),
    })
    return
}

```

### 函数做两件事

1. **读取请求 Body**：读取 POST/PUT 请求体内的 JSON 数据；
2. **绑定 + 自动校验**：
   - 根据`json:"xxx"`标签，把 json 的值赋值给结构体`req`；
   - 立刻执行`binding`里面全部校验规则。

  3.ShouldBindJSON()必须传指针

### 返回err两种情况

1. **JSON 格式错误**：前端传的不是合法 JSON；
2. **校验不通过**：JSON 格式没问题，但是数据不符合 binding 规则，例如 name 为空、邮箱格式错误。

- `err != nil`：参数异常，返回 400 给前端，`return`终止，**不往下执行业务逻辑**。
- `err == nil`：绑定、校验全部成功，`req`结构体里面已经拿到前端提交的数据，可以直接使用：`req.Name`、`req.Email`。

注意：`ShouldBindJSON` 只读取请求 Body 里的 JSON，不读取 URL 查询参数、路径参数。

### 3. `validationError(err error)`：解析校验错误

​      validator 原生错误全部是英文，这个函数把错误提取，组装成 `map[string]string`，返回给前端友好提示；

```go
func validationError(err error) map[string]string {
    errors := make(map[string]string)
    var ve validator.ValidationErrors
    if errors.As(err, &ve) {
        for _, fe := range ve {
            errors[fe.Field()] = msgForTag(fe.Tag())
        }
    }
    return errors
}

```

1. `var ve validator.ValidationErrors`：validator 定义的校验错误切片类型；
2. `errors.As(err, &ve)`：把原始 err 断言转换为校验错误类型。


如果是 JSON 格式错误，err 不能转为`ve`，`errors`返回空 map。

3. 循环遍历每一个字段错误对象 `fe`：
   - `fe.Field()`：**结构体字段名**，如`Name`、`Email`
   - `fe.Tag()`：触发了哪一条校验规则，如`required`、`email`、`min`

### 4.Gin不同位置参数对应不同绑定方法

前端参数有三个位置，使用不同绑定函数：

| 参数位置           | 示例 url                           | 绑定函数                  | 结构体 tag    |
| ------------------ | ---------------------------------- | ------------------------- | ------------- |
| **Body JSON**      | `POST /user` body: `{"name":"xx"}` | `c.ShouldBindJSON(&req)`  | `json:"name"` |
| **查询参数 Query** | `/list?name=zhangsan&age=10`       | `c.ShouldBindQuery(&req)` | `form:"name"` |
| **路径参数 Uri**   | `/user/:id`                        | `c.ShouldBindUri(&req)`   | `uri:"id"`    |

举例完整流程梳理（POST新增用户）

1. 前端 axios POST 发送 JSON 到`/user`；
2. Gin 路由匹配，执行`createUser` handler；
3. `var req CreateUserRequest` 声明结构体；
4. `c.ShouldBindJSON(&req)`：读取 body → 解析 json 赋值结构体 →执行 binding 校验；
5. 如果出错：解析 validator 错误，返回 400 错误 json，return，业务逻辑不执行；
6. 如果成功：req 拿到全部前端参数，执行业务逻辑（数据库新增用户）；
7. 返回`201 Created`成功响应。

## 4.响应格式

后端接口返回给前端的 JSON，**全部固定一套格式**，就叫统一响应结构。
前端写代码的时候只需要解析固定字段 `code`、`message`、`data`，不用适配五花八门的返回格式。

```
// 统一响应结构体
type Response struct {
    Code    int         `json:"code"` //状态码
    Message string      `json:"message"`//提示信息
    Data    interface{} `json:"data,omitempty"`//`omitempty` 非常重要：如果                                                    //`Data` 为 `nil`，序列化 JSON 的时候                                                //直接忽略这个字段，不输出 data
}
```

#### 两个工具函数

```
// 成功响应
func Success(c *gin.Context, data interface{}) {
    c.JSON(http.StatusOK, Response{
        Code:    0,
        Message: "success",
        Data:    data,
    })
}
```

`http.StatusOK = 200` HTTP 状态码；

`Code=0` 业务成功；

参数`data interface{}`：要返回给前端的数据，可以传结构体、gin.H、切片、nil。

1：返回用户对象

```go
// 校验通过后，业务逻辑
user := gin.H{"id":1,"name":"张三"}
Success(c, user)

```

2:前端收到JSON

```go
{
    "code": 0,
    "message": "success",
    "data": {
        "id": 1,
        "name": "张三"
    }
}
```



```go
func Error(c *gin.Context, httpStatus int, message string) {
    c.JSON(httpStatus, Response{
        Code:    httpStatus,
        Message: message,
    })
}

```

httpStatus：HTTP 响应状态码，比如 400、401、500；同时赋值给业务 Code。

message：错误提示文本。

Data 没有赋值，默认 nil，`omitempty`会把 data 字段从 json 中删掉

**示例**

鉴权失败，401

```go
Error(c, http.StatusUnauthorized, "token缺失，请登录")
```

HTTP 响应状态码：401，返回 JSON：

```go
{
    "code": 401,
    "message": "token缺失，请登录"
}
```

参数校验失败：

```go
Error(c, http.StatusBadRequest, "参数校验失败")
```





### 整体流程回顾

1. 前端发请求；
2. 路由匹配，执行 handler；
3. ShouldBindJSON 绑定校验；
4. 校验失败：用 Response 结构体返回错误信息；
5. 校验成功执行业务逻辑；
6. 使用 Success 返回业务数据。

总结一下：统一响应就是定义固定结构体，封装Success,Error工具函数，所有接口都用这套结构体返回JSON,前端统一解析code/message/data

```go
package main

import (
	"errors"
	"github.com/gin-gonic/gin"
	"github.com/go-playground/validator/v10"
	"net/http"
)

// -------------------------- 统一响应结构 --------------------------
type Response struct {
	Code    int         `json:"code"`
	Message string      `json:"message"`
	Data    interface{} `json:"data,omitempty"`
}

// Success 成功响应
func Success(c *gin.Context, data interface{}) {
	c.JSON(http.StatusOK, Response{
		Code:    0,
		Message: "success",
		Data:    data,
	})
}

// Error 失败响应
func Error(c *gin.Context, httpStatus int, message string) {
	c.JSON(httpStatus, Response{
		Code:    httpStatus,
		Message: message,
	})
}

// -------------------------- 参数校验错误解析 --------------------------
func validationError(err error) map[string]string {
	errMap := make(map[string]string)
	var ve validator.ValidationErrors
	if errors.As(err, &ve) {
		for _, fe := range ve {
			errMap[fe.Field()] = msgForTag(fe.Tag())
		}
	}
	return errMap
}

func msgForTag(tag string) string {
	switch tag {
	case "required":
		return "该字段为必填项"
	case "email":
		return "邮箱格式不正确"
	case "min":
		return "长度或者数值过小"
	case "max":
		return "长度或者数值过大"
	case "gte":
		return "数值不能小于最小值"
	case "lte":
		return "数值不能大于最大值"
	default:
		return "参数非法"
	}
}

// -------------------------- 请求结构体（参数验证） --------------------------
// 创建用户
type CreateUserRequest struct {
	Name     string `json:"name" binding:"required,min=2,max=50"`
	Email    string `json:"email" binding:"required,email"`
	Age      int    `json:"age" binding:"gte=0,lte=150"`
	Password string `json:"password" binding:"required,min=8"`
}

// 更新用户 PUT全量更新
type UpdateUserRequest struct {
	Name     string `json:"name" binding:"required,min=2,max=50"`
	Email    string `json:"email" binding:"required,email"`
	Age      int    `json:"age" binding:"gte=0,lte=150"`
	Password string `json:"password" binding:"required,min=8"`
}

// -------------------------- 模拟数据库（内存存储） --------------------------
type User struct {
	ID       int    `json:"id"`
	Name     string `json:"name"`
	Email    string `json:"email"`
	Age      int    `json:"age"`
	Password string `json:"-"` // json:"-" 序列化忽略密码，不返回前端
}

var (
	userList []User
	nextID   = 1
)

// -------------------------- CRUD Handler --------------------------

// CreateUser POST /users 创建用户
func CreateUser(c *gin.Context) {
	var req CreateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		errMap := validationError(err)
		if len(errMap) > 0 {
			c.JSON(http.StatusBadRequest, Response{
				Code:    http.StatusBadRequest,
				Message: "参数校验失败",
				Data:    errMap,
			})
			return
		}
		Error(c, http.StatusBadRequest, "JSON格式错误")
		return
	}

	newUser := User{
		ID:       nextID,
		Name:     req.Name,
		Email:    req.Email,
		Age:      req.Age,
		Password: req.Password,
	}
	nextID++
	userList = append(userList, newUser)

	Success(c, newUser)
}

// GetUserList GET /users 获取全部用户列表
func GetUserList(c *gin.Context) {
	Success(c, userList)
}

// GetUser GET /users/:id 获取单个用户
func GetUser(c *gin.Context) {
	id := c.Param("id")
	// 简易转int
	var uid int
	_, err := gin.H{}.MapJSON(id, &uid)
	if err != nil {
		Error(c, http.StatusBadRequest, "id必须为数字")
		return
	}

	var target *User
	for _, u := range userList {
		if u.ID == uid {
			tmp := u
			target = &tmp
			break
		}
	}
	if target == nil {
		Error(c, http.StatusNotFound, "用户不存在")
		return
	}
	Success(c, target)
}

// UpdateUser PUT /users/:id 全量更新用户
func UpdateUser(c *gin.Context) {
	var req UpdateUserRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		errMap := validationError(err)
		if len(errMap) > 0 {
			c.JSON(http.StatusBadRequest, Response{
				Code:    http.StatusBadRequest,
				Message: "参数校验失败",
				Data:    errMap,
			})
			return
		}
		Error(c, http.StatusBadRequest, "JSON格式错误")
		return
	}

	idStr := c.Param("id")
	var uid int
	_, err := gin.H{}.MapJSON(idStr, &uid)
	if err != nil {
		Error(c, http.StatusBadRequest, "id必须数字")
		return
	}

	// 查找索引
	idx := -1
	for i, u := range userList {
		if u.ID == uid {
			idx = i
			break
		}
	}
	if idx == -1 {
		Error(c, http.StatusNotFound, "用户不存在")
		return
	}

	// 全量覆盖更新
	userList[idx].Name = req.Name
	userList[idx].Email = req.Email
	userList[idx].Age = req.Age
	userList[idx].Password = req.Password

	Success(c, userList[idx])
}

// DeleteUser DELETE /users/:id 删除用户
func DeleteUser(c *gin.Context) {
	idStr := c.Param("id")
	var uid int
	_, err := gin.H{}.MapJSON(idStr, &uid)
	if err != nil {
		Error(c, http.StatusBadRequest, "id必须数字")
		return
	}

	idx := -1
	for i, u := range userList {
		if u.ID == uid {
			idx = i
			break
		}
	}
	if idx == -1 {
		Error(c, http.StatusNotFound, "用户不存在")
		return
	}

	// 删除切片元素
	userList = append(userList[:idx], userList[idx+1:]...)
	Success(c, gin.H{"msg": "删除成功"})
}

func main() {
	r := gin.Default() // 内置全局中间件 Logger + Recovery

	// RESTful 用户路由组
	userGroup := r.Group("/users")
	{
		userGroup.POST("", CreateUser)       // 创建用户 POST /users
		userGroup.GET("", GetUserList)       // 查询列表 GET /users
		userGroup.GET("/:id", GetUser)       // 查询单个 GET /users/1
		userGroup.PUT("/:id", UpdateUser)    // 更新 PUT /users/1
		userGroup.DELETE("/:id", DeleteUser) // 删除 DELETE /users/1
	}

	r.Run(":8080")
}

```

