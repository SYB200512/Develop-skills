# GO：中间件开发

## 中间件固定模板

```go
func 中间件名(可选参数) gin.HandlerFunc {
    // 【1. 初始化阶段：只执行一次！服务启动的时候执行】
    // 适合创建对象、初始化变量（限流的limiter在这里创建）

    // 返回一个匿名函数，类型 gin.HandlerFunc
    return func(c *gin.Context) {
        // 【2. 请求前置逻辑：c.Next() 之前】
        // 请求进来，先跑这里。做校验、设置header、记录开始时间

        c.Next() // ✅ 关键！放行，执行链条后面的中间件/handler

        // 【3. 请求后置逻辑：c.Next() 之后】
        // 后面全部中间件 + handler 全部执行完，**再回来执行这里**
    }
}

```

## 1.日志中间件

```
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()                 // 前置：记录请求开始时间
        path := c.Request.URL.Path          // 前置：拿到请求路径

        c.Next()                            // 放行，去执行后面所有逻辑

        // ✅ 下面全部是后置代码：业务handler跑完之后才执行！
        latency := time.Since(start)        // 计算耗时
        status := c.Writer.Status()         // 获取最终返回的HTTP状态码
        log.Printf("[%d] %s %s %v",
            status,
            c.Request.Method,
            path,
            latency,
        )
    }
}

```

1. `start := time.Now()`（前置）
2. `c.Next()` 进入下一环（其他中间件 + 你的接口 handler）
3. handler 执行完毕，回到 Logger，执行**后置日志打印**
4. 返回响应给前端

>
> 特点：**用来记录请求耗时、响应状态，属于后置处理**

## 2.CORS中间件

CORS中间件就是在响应里添加CORS相关的HTTP响应头，告诉浏览器：允许这个前端域名跨域访问我的接口

浏览器在非简单请求前，会先发OPTIONS预检请求，询问服务器是否允许跨域

- OPTIONS请求不需要执行业务代码，直接返回204，c.AbortWithStatus(204)终止链条
- 所有跨域Header必须在前置阶段设置（c.Next()之前），因为Header要在响应发送前写入
- CORS中间件必须放在最前面，优先处理OPTIONS预检请求

```go
func CORS(allowedOrigins []string) gin.HandlerFunc {
    return func(c *gin.Context) {
        origin := c.Request.Header.Get("Origin")

        // 判断来源域名是否允许
        for _, allowed := range allowedOrigins {
            if origin == allowed || allowed == "*" {
                c.Header("Access-Control-Allow-Origin", origin)
                break
            }
        }
        // 设置跨域响应头
        c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Content-Type, Authorization")
        c.Header("Access-Control-Max-Age", "86400")

        // 浏览器预检请求 OPTIONS
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(204)
            return
        }

        c.Next()
    }
}

```

## 3.限流中间件RateLimit

命令桶：golang.org/x/time/rate：系统按照固定速率向桶里放令牌，桶里有最大容量burst，满了后拒绝访问

```go
func RateLimit(rate int, burst int) gin.HandlerFunc {
    // ========= 【初始化阶段，只运行一次！服务启动时执行】=========
    limiter := rate.NewLimiter(rate.Limit(rate), burst)
    // rate：每秒允许多少请求
    // burst：令牌桶最大容量（突发请求上限）

    return func(c *gin.Context) {
        // ========= 请求前置逻辑 =========
        if !limiter.Allow() { // 判断有没有令牌
            // 没有令牌，请求太多，429 Too Many Requests
            c.AbortWithStatusJSON(429, gin.H{
                "error": "too many requests",
            })
            return
        }
        c.Next() // 拿到令牌，放行
    }
}
```

示例：`RateLimit(10,20)`

- 正常：每秒最多稳定处理 10 个请求
- 突发：一瞬间最多可以一次性处理 20 个请求（桶存满 20 个令牌）
- 超过 20 个的请求直接 429