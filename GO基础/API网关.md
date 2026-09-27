# API网关

## 一、API 网关是什么？

API 网关是微服务架构中的 **统一入口**。客户端不直接访问各个微服务，而是先访问网关，由网关完成：

- 路由转发
- 认证鉴权
- 限流熔断
- 协议转换
- 日志监控
- 灰度发布
- 请求聚合
- 缓存、压缩、CORS 等

你可以把它理解成：

> **反向代理 + 路由中心 + 认证中心 + 流量治理中心 + 可观测性入口**

架构位置大致是：

```text
客户端 / 浏览器 / App / 第三方
        |
        v
    负载均衡 LB
        |
        v
     API 网关
        |
        +--> 用户服务 user-service
        +--> 订单服务 order-service
        +--> 支付服务 pay-service
        +--> 商品服务 product-service
```

它屏蔽了内部微服务的拓扑、地址、协议差异，让客户端只需要知道一个入口。

## 二、API 网关和 Nginx、负载均衡、服务网格的区别

| 概念                  | 主要职责                            | 和 API 网关的关系                |
| :-------------------- | :---------------------------------- | :------------------------------- |
| Nginx                 | 反向代理、负载均衡、静态资源、SSL   | 可以做简易网关，但动态治理能力弱 |
| 负载均衡 LB           | 流量分发到多个实例                  | 通常位于网关前面                 |
| API 网关              | 路由、认证、限流、协议转换、监控    | 业务流量统一入口                 |
| 服务网格 Service Mesh | 服务间通信治理，如 mTLS、熔断、追踪 | 偏东西向流量，网关偏南北向流量   |
| BFF                   | 面向前端的聚合层                    | 可以放在网关之后，为不同端定制   |

一句话：

- **南北向流量**：客户端 → 网关 → 内部服务。
- **东西向流量**：服务 → 服务，常由服务网格或 RPC 框架处理。

## 三、核心职责详解

### 1. 路由转发

按路径、Host、Header、Query、Method、权重等把请求转发到后端服务。

例如：

```text
/api/users/**   -> user-service:8080
/api/orders/**  -> order-service:8080
/api/pay/**     -> pay-service:8080
```

进阶能力：

- 路径重写：`/api/users/123` → `/users/123`
- 版本路由：`/v1/users`、`/v2/users`
- 灰度发布：10% 流量到新版本
- 权重路由：按比例分发
- 蓝绿发布、AB 测试
- 根据 Header 路由：`X-App-Version: 2.0`

你笔记里的：

```go
http.HandleFunc("/api/users/", func(w http.ResponseWriter, r *http.Request) {
    reverseProxy("user-service:8080").ServeHTTP(w, r)
})
```

就是最简单的路径路由：所有 `/api/users/` 开头的请求都转发到 `user-service:8080`。

------

### 2. 认证鉴权

网关统一做认证，后端服务只关心业务。常见方式：

- JWT
- OAuth2 / OIDC
- API Key
- Session + Redis
- mTLS
- RBAC / ABAC 权限模型

典型流程：客户端带 `Authorization: Bearer xxx`。

2. 网关校验 Token。

3. 解析出用户 ID、角色、租户 ID。

4. 写入内部 Header，例如：

   ```text
   X-User-ID: 123
   X-Role: admin
   X-Tenant-ID: t001
   ```

5. 转发给下游服务。

注意：

网关必须删除外部伪造的 `X-User-ID`，再写入可信值。下游服务不能直接相信客户端传来的内部 Header。

### 3. 限流熔断

这是网关最核心的稳定性能力。

#### 限流

常见维度：

- IP
- 用户 ID
- API 路径
- 租户
- 应用 AppID

常见算法：

| 算法     | 特点                 |
| :------- | :------------------- |
| 固定窗口 | 简单，有临界突刺问题 |
| 滑动窗口 | 更平滑               |
| 漏桶     | 恒定速率输出         |
| 令牌桶   | 允许一定突发，最常用 |

单机限流可以用：

```go
limiter := rate.NewLimiter(rate.Limit(100), 200)
if !limiter.Allow() {
    http.Error(w, "too many requests", http.StatusTooManyRequests)
    return
}
```

分布式限流通常用 Redis + Lua，或者 Sentinel、Envoy Rate Limit。

#### 熔断

当下游服务错误率过高时，网关快速失败，避免雪崩。

熔断器状态：

```text
Closed：正常放行
Open：直接失败，不请求下游
Half-Open：放少量请求试探，成功则恢复 Closed
```

常见库：`sony/gobreaker`。

#### 超时、重试、降级

- 超时：防止请求堆积。
- 重试：只对幂等请求重试，否则可能重复下单。
- 降级：返回缓存、默认值、友好错误。

一般原则：

> 网关超时时间略大于下游服务超时时间。

### 4. 协议转换

网关可以屏蔽协议差异：

- REST/JSON ↔ gRPC/Protobuf
- HTTP/1.1 ↔ HTTP/2
- WebSocket 代理
- MQ 转 HTTP
- 外部 JSON → 内部 Protobuf

例如客户端发：

```http
POST /api/users
Content-Type: application/json

{"name":"Tom"}
```

网关转成 gRPC 调用：

```proto
rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
```

Go 里常用 `grpc-gateway` 做 REST ↔ gRPC 转换。

### 5. 日志监控

网关是全链路入口，最适合做统一观测。

需要记录：

- 请求方法、路径、状态码、耗时
- 请求 ID / Trace ID
- 用户 ID、租户 ID
- 下游服务、实例 IP
- 错误信息
- QPS、延迟、错误率、饱和度

常用方案：

- 日志：Zap、Logrus
- 指标：Prometheus + Grafana
- 链路追踪：OpenTelemetry + Jaeger
- 告警：Alertmanager

请求 ID 示例：

```go
requestID := r.Header.Get("X-Request-ID")
if requestID == "" {
    requestID = uuid.NewString()
}
r.Header.Set("X-Request-ID", requestID)
w.Header().Set("X-Request-ID", requestID)
```

### 6. 其他常见能力

- SSL 终止
- CORS 跨域
- Gzip / Brotli 压缩
- 响应缓存
- 请求体大小限制
- IP 黑白名单
- WAF 防护
- 防重放、防篡改
- 参数校验
- 请求聚合 / BFF
- 配额计费
- 灰度发布
- 动态配置热更新

## 四、一次请求完整流程

```text
1. 客户端请求 https://api.example.com/api/users/123
2. DNS 解析到负载均衡
3. LB 转发到某个 API 网关实例
4. 网关生成/透传 Request-ID
5. 路由匹配：/api/users/** -> user-service
6. 认证：校验 JWT
7. 鉴权：是否有权限访问该接口
8. 限流：是否超过 QPS
9. 熔断检查：下游是否已熔断
10. 路径重写、Header 清洗
11. 反向代理到 user-service:8080
12. 下游返回响应
13. 网关记录日志、指标、Trace
14. 响应客户端
```

## 五、方案对比

| 方案                 | 基础             | 性能       | 配置复杂度 | 适用场景             |
| :------------------- | :--------------- | :--------- | :--------- | :------------------- |
| Nginx / OpenResty    | C + Lua          | 高         | 中高       | 传统网关、自定义 Lua |
| Kong                 | OpenResty        | 中高       | 中         | 企业级、插件丰富     |
| APISIX               | OpenResty + etcd | 高         | 中         | 云原生、动态路由     |
| Traefik              | Go               | 高         | 低         | K8s 原生、自动发现   |
| Envoy                | C++              | 高         | 高         | 服务网格、xDS        |
| Spring Cloud Gateway | Java             | 中         | 低         | Java 微服务生态      |
| 自研 Go              | Go               | 取决于实现 | 高         | 定制化需求           |

## 六、代码示例

```go
package main

import (
    "log"
    "net/http"
    "net/http/httputil"
    "net/url"
    "time"
)
//把字符串 URL 转成 *url.URL 对象。
func mustURL(raw string) *url.URL {
    u, err := url.Parse(raw)
    if err != nil {
        panic(err)
    }
    return u
}

//创建反向代理
//设置错误处理
//设置http连接池
func newProxy(target *url.URL) *httputil.ReverseProxy {
    proxy := httputil.NewSingleHostReverseProxy(target)

    proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, err error) {
        log.Printf("proxy error: %v", err)
        http.Error(w, "bad gateway", http.StatusBadGateway)
    }

    proxy.Transport = &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20,
        IdleConnTimeout:     90 * time.Second,
    }

    return proxy
}

func main() {
    routes := map[string]*url.URL{
        "/api/users/":  mustURL("http://user-service:8080"),
        "/api/orders/": mustURL("http://order-service:8080"),
    }

    for prefix, target := range routes {
        proxy := newProxy(target)

        http.Handle(prefix, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 这里可以加：认证、限流、日志、Trace
            proxy.ServeHTTP(w, r)
        }))
    }

    log.Fatal(http.ListenAndServe(":8000", nil))
}
```

