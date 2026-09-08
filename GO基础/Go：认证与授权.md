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





# 整合 JWT + RBAC 完整流程回顾

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