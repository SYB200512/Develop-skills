# Go：认证与授权

## 1.JWT认证实现

JWT（JSON‑Web‑Token）：登录成功后，服务端生成一段加密字符串返回给前端；
前端后续每次请求在请求头 `Authorization` 带上这个 token，服务端校验 token，判断用户是否登录。

流程：

1. 用户登录 → 账号密码校验成功 → 服务端生成 JWT 返回前端
2. 前端存 token（localStorage /cookie）
3. 访问受保护接口：请求头带上 `Authorization: Bearer xxx.token.xxx`
4. JWT 中间件拦截请求，校验 token；校验通过 `c.Next()`放行接口；失败 `c.AbortWithStatusJSON`直接终止请求链条

```go
// JWT 中间件
func JWTAuth(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        tokenString := c.GetHeader("Authorization")
        if tokenString == "" || !strings.HasPrefix(tokenString, "Bearer ") {
            c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
            return
        }

        tokenString = strings.TrimPrefix(tokenString, "Bearer ")
        claims := &Claims{}
        token, err := jwt.ParseWithClaims(tokenString, claims, func(t *jwt.Token) (interface{}, error) {
            return []byte(secret), nil
        })

        if err != nil || !token.Valid {
            c.AbortWithStatusJSON(401, gin.H{"error": "invalid token"})
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("role", claims.Role)
        c.Next()
    }
}
```

1. 请求进来，进入 JWTAuth 中间件
2. 获取 Header `Authorization`，校验`Bearer `前缀
3. 剥离前缀得到 token 原始字符串
4. `jwt.ParseWithClaims` 使用 secret 解析、验签、校验过期
5. 校验通过：`c.Set()`把用户信息存入上下文
6. `c.Next()`放行，执行受保护接口 handler；handler 中`c.Get()`拿到 user_id 做业务。

## 2.RBAC权限模型

1. 用户不直接分配权限；
2. 给用户分配**角色**（`admin`、`editor`、`viewer`）；
3. 给角色配置一堆权限：资源 + 操作；
4. 判断权限的时候：拿到用户角色，查这个角色有没有对该资源做该操作的权限。

```go
// 数据库模型
type User struct {
    ID   uint   `gorm:"primaryKey"`
    Role string `gorm:"size:50;not null"` // admin, editor, viewer
}

//用户表存`Role`字段，一个用户对应一个角色
//JWT 登录的时候，会把这个`Role`放进 token 的 Claims；
//JWT 中间件解析 token 后执行 `c.Set("role", claims.Role)`，存到 gin 上下文。

// 权限检查
type Permission struct {
    Resource string   // "article", "user", "system"
    Actions  []string // "create", "read", "update", "delete"
}

//`Resource`：**资源**，你要操作哪个模块；
//`Actions`：**允许的操作集合**



var rolePermissions = map[string][]Permission{
    "admin": {
        {Resource: "article", Actions: []string{"create", "read", "update", "delete"}},
        {Resource: "user", Actions: []string{"create", "read", "update", "delete"}},
    },
    "editor": {
        {Resource: "article", Actions: []string{"create", "read", "update"}},
        {Resource: "user", Actions: []string{"read"}},
    },
    "viewer": {
        {Resource: "article", Actions: []string{"read"}},
    },
}



// 权限中间件
func RequirePermission(resource string, action string) gin.HandlerFunc {
    return func(c *gin.Context) {
        role := c.GetString("role")
        permissions := rolePermissions[role]

        for _, p := range permissions {
            if p.Resource == resource {
                for _, a := range p.Actions {
                    if a == action {
                        c.Next()
                        return
                    }
                }
            }
        }

        c.AbortWithStatusJSON(403, gin.H{"error": "forbidden"})
    }
//1. 遍历角色每一条权限`p`；匹配目标`resource`资源。
//2. 在该资源下遍历允许的操作`Actions`；
//3. 如果找到需要的`action`：说明有权限。
  //- `c.Next()`：放行，执行后续 handler 业务代码。
   //- `return`：结束当前中间件函数。
    
    
}
```





# 3.整合 JWT + RBAC 完整流程回顾

1. 用户登录：校验账号密码，读取用户`role`；把`user_id`、`role`写入 JWT Claims；返回 token。
2. 前端请求接口，Header 带上`Authorization: Bearer token`。
3. 请求到达 Gin：
   ① JWTAuth 中间件：解析 token。
   - 失败：401；Abort 终止。
   - 成功：`c.Set("user_id")`、`c.Set("role")`；`c.Next()`。
   ② RequirePermission 权限中间件：读取上下文 role，校验是否拥有【资源 + 操作】权限。
   - 无权限：403；Abort 终止。
   - 有权限：`c.Next()`放行。
   ③ 执行业务 handler。



## 4.练习

#### 实现 JWT 的 Access Token + Refresh Token 双 Token 机制。

##### 前提知识：

单 JWT 问题：

1. 如果 AccessToken 有效期设置太长（比如 7 天），一旦 token 泄露，攻击者可以一直用，无法立刻失效。
2. 如果 AccessToken 有效期很短（比如 15 分钟），用户频繁登录，体验很差。

双 Token 思路：

- AccessToken（访问令牌）：短期有效，例如 15 分钟。用来访问业务接口，放在请求头`Authorization: Bearer xxx`。过期后不能访问接口。
- RefreshToken（刷新令牌）：长期有效，例如 7 天。不能访问业务接口，专门用来换取新的 AccessToken。

##### 流程：

1. 用户登录成功，同时返回 AccessToken + RefreshToken
2. 前端用 AccessToken 请求业务接口；15 分钟后 AccessToken 过期
3. 前端拿着 RefreshToken 调用 `/refresh-token` 接口，换取新 AccessToken
4. 如果 RefreshToken 过期 / 失效，才强制用户重新登录

```go
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/golang-jwt/jwt/v5"
	"net/http"
	"time"
)

// 密钥
const secret = "my-secret-key"

// AccessToken 载荷：业务鉴权需要的信息
type AccessClaims struct {
	UserID uint   `json:"user_id"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

// RefreshToken 载荷：只存userID
type RefreshClaims struct {
	UserID uint `json:"user_id"`
	jwt.RegisteredClaims
}

// 生成AccessToken 短期：15分钟
func genAccess(userID uint, role string) (string, error) {
	claims := AccessClaims{
		UserID: userID,
		Role:   role,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
		},
	}
	t := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return t.SignedString([]byte(secret))
}

// 生成RefreshToken 长期：7天
func genRefresh(userID uint) (string, error) {
	claims := RefreshClaims{
		UserID: userID,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(7 * 24 * time.Hour)),
		},
	}
	t := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return t.SignedString([]byte(secret))
}

// JWT中间件：校验AccessToken，业务接口使用
func JWTAuth() gin.HandlerFunc {
	return func(c *gin.Context) {
		tokenStr := c.GetHeader("Authorization")
		if tokenStr == "" || len(tokenStr) < 7 || tokenStr[:7] != "Bearer " {
			c.JSON(http.StatusUnauthorized, gin.H{"msg": "未登录"})
			c.Abort()
			return
		}
		tokenStr = tokenStr[7:]

		claims := &AccessClaims{}
		token, err := jwt.ParseWithClaims(tokenStr, claims, func(t *jwt.Token) (interface{}, error) {
			return []byte(secret), nil
		})
		if err != nil || !token.Valid {
			c.JSON(http.StatusUnauthorized, gin.H{"msg": "token失效"})
			c.Abort()
			return
		}
		// 存入上下文
		c.Set("userID", claims.UserID)
		c.Set("role", claims.Role)
		c.Next()
	}
}

// 登录接口：返回 access + refresh
func login(c *gin.Context) {
	// 模拟账号密码校验
	var req struct {
		Username string `json:"username"`
		Password string `json:"password"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"msg": "参数错误"})
		return
	}
	if req.Username == "admin" && req.Password == "123456" {
		userID := uint(1)
		role := "admin"
		access, _ := genAccess(userID, role)
		refresh, _ := genRefresh(userID)
		c.JSON(http.StatusOK, gin.H{
			"access_token":  access,
			"refresh_token": refresh,
		})
		return
	}
	c.JSON(http.StatusUnauthorized, gin.H{"msg": "账号密码错误"})
}

// 刷新token接口：传入refresh，换取新access
func refreshToken(c *gin.Context) {
	var req struct {
		RefreshToken string `json:"refresh_token"`
	}
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"msg": "参数错误"})
		return
	}

	claims := &RefreshClaims{}
	token, err := jwt.ParseWithClaims(req.RefreshToken, claims, func(t *jwt.Token) (interface{}, error) {
		return []byte(secret), nil
	})
	if err != nil || !token.Valid {
		c.JSON(http.StatusUnauthorized, gin.H{"msg": "refresh失效，请重新登录"})
		return
	}

	// 拿到用户ID，模拟查库获取角色
	userID := claims.UserID
	role := "admin"
	newAccess, _ := genAccess(userID, role)

	c.JSON(http.StatusOK, gin.H{
		"access_token": newAccess,
	})
}

// 受保护接口，测试AccessToken
func getUserInfo(c *gin.Context) {
	userID, _ := c.Get("userID")
	role, _ := c.Get("role")
	c.JSON(http.StatusOK, gin.H{"userID": userID, "role": role})
}

func main() {
	r := gin.Default()

	// 公开接口
	r.POST("/login", login)
	r.POST("/refresh", refreshToken)

	// 需要AccessToken鉴权的路由
	api := r.Group("/api")
	api.Use(JWTAuth())
	{
		api.GET("/userinfo", getUserInfo)
	}
	r.Run(":8080")
}

```

1. POST `/login` 登录成功，返回 `access_token` + `refresh_token`
2. 前端用 `access_token` 请求 `/api/userinfo`
3. 15 分钟后 access 过期，访问业务接口会返回 token 失效
4. 前端捕获 401，调用 POST `/refresh`，带上`refresh_token`
5. 后端校验 refresh 合法，生成**新 access_token**返回
6. 前端拿到新 access，继续访问业务接口
7. 如果 refresh 也过期了 → 强制跳登录页