# 分布式追踪

## 一、背景：为什么需要分布式追踪

单体应用时代，所有代码在一个进程里。出问题看日志，打印调用耗时，很容易定位慢代码、报错位置。

**微服务 / 分布式系统**：一次前端请求，会依次调用网关、订单服务、支付服务、数据库、消息队列，跨多个进程、多台机器。 问题痛点：

1. 一个请求散落在多个服务日志里，日志没有统一标识，很难把整条链路串起来；
2. 不知道哪一段调用耗时最长，无法定位慢节点；
3. 调用失败时，分不清是网关、订单、支付还是数据库的问题。

> **分布式追踪就是用来记录一次请求在各个服务之间完整调用链路的技术**，OpenTelemetry（OTel）是当前业界标准的可观测性 SDK，用来采集 Trace 数据。

## 二、核心概念

### 1. Span（跨度）

**Span：链路里最小的工作单元，表示一次独立操作。** 对应一次函数调用、一次 HTTP 请求、一次 SQL 查询、一次 Kafka 消息发送。

> 你笔记里例子：
>
> - POST /api/order（网关收到请求，是一个 Span）
> - Validate Order（订单服务校验订单，一个 Span）
> - Create Order (DB)（写数据库，一个 Span）
> - Publish Event (Kafka)（发消息，一个 Span）
> - Process Payment（支付处理，一个 Span）

每个 Span 自带这些信息：

1. **Span ID**：当前这个操作的唯一 ID；
2. 开始时间、结束时间 → 计算耗时；
3. 操作名称（operation name）；
4. 属性（attributes）：HTTP 状态码、db 地址、方法名；
5. 事件（events）：中间发生的日志、异常；
6. 状态（status）：成功 / 失败。

Span 之间有父子关系：

- 父 Span：发起调用的操作
- 子 Span：被调用的操作

> 网关收到请求的 Span 是父 Span；网关调用订单服务，订单服务里面的校验、入库 Span 都是它的子 Span。

### 2. Trace（追踪链路）

**Trace：由一组有父子关系的 Span 组成的完整调用链，代表一次端到端请求。** 整条 Trace 拥有唯一的 **Trace ID**。

> 例子：用户下单这个请求，从网关 → 订单服务 → 支付服务，所有相关 Span 合在一起，就是一条 Trace，共用同一个 Trace-ID。

✅ 一句话区分：

- **Trace = 整条请求链路（一个 TraceID）**
- **Span = 链路里每一小步操作（各自 SpanID）**

### 3. Context Propagation 上下文透传（非常关键）

> 你笔记中标注：`Context Propagation (Trace-ID, Span-ID)`

Span 在不同服务（不同进程 / 机器），怎么把 TraceID、父 SpanID 传给下游服务？ **上下文透传：把追踪信息放在请求载体里传递给下游。**

常见载体：

- HTTP：放在请求 Header（如 `traceparent`，OTel 标准 header）
- gRPC：放在 metadata
- Kafka/RabbitMQ 消息：放在消息 header

流程：

1. 网关收到请求，生成 TraceID，创建根 Span；
2. 网关调用订单服务，在 HTTP Header 带上 `traceparent`（TraceID + 父 SpanID）；
3. 订单服务读取 header，**继承 TraceID**，创建子 Span；
4. 订单服务继续调用支付服务，继续把 Trace 信息往下传；
5. 所有服务上报各自 Span 数据到追踪后端（Jaeger、Zipkin）；
6. 后端根据 TraceID，把所有 Span 拼接成完整调用链可视化。

> 如果不做透传：下游服务会新建一条独立 Trace，链路就断了，无法串联。

## 三、OpenTelemetry（OTel）介绍

> OpenTelemetry（简称 OTel）：**一套厂商无关的可观测性 SDK 规范与实现**，用于生成、采集、导出追踪 (Trace)、指标 (Metrics)、日志 (Logs) 数据。 OTel 本身**不存储、不展示**链路数据；它只负责埋点、生成 Span，把数据通过 OTLP 协议推送到后端（Jaeger、Zipkin、OTel Collector），再由后端做可视化查询。

## OTel 核心组件梳理

1. **TracerProvider**：追踪器提供者，全局顶层管理器。配置导出器、批处理、资源信息。一个应用只需要一个。

2. **Tracer**：由 TracerProvider 创建，用来启动 Span。一个服务可以有多个 Tracer（按模块划分）。

3. **Span**：链路最小单元，由 Tracer.Start () 创建。必须调用`span.End()`结束，否则不会上报。

4. **Context（上下文）**：Go 里`context.Context`，用来携带 TraceID、父 SpanID，**跨函数、跨服务透传追踪信息**。

5. Exporter（导出器）

   ：负责把 Span 数据发出去。

   - `otlptracehttp`：使用 OTLP over HTTP 协议推送 Trace（示例代码用这个）
   - `otlptracegrpc`：OTLP over gRPC，性能更好，生产常用

6. **Batcher（批处理器）**：`sdktrace.WithBatcher`，不会每产生一个 Span 就立刻网络发送；攒一批 Span 批量上报，降低网络开销。

7. **Resource（资源）**：描述当前服务的元信息：服务名称、服务版本、实例 ID 等，用`semconv`（语义约定包）标准化字段。

8. **Propagator（传播器）**：负责把 Trace 信息序列化到 HTTP Header（`traceparent`），下游服务从 Header 解析，实现跨服务上下文透传。默认使用 W3C traceparent 规范

## 完整代码（服务 A：http 服务，自动埋点，手动创建子 Span）

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"go.opentelemetry.io/otel/semconv/v1.26.0"
	"go.opentelemetry.io/otel/trace"

	// http自动埋点中间件，自动创建span、自动处理traceparent header透传
	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

// initTracer 初始化OTel追踪，返回TracerProvider，用于程序关闭时优雅shutdown
func initTracer() (*sdktrace.TracerProvider, error) {
	// 1. 创建OTLP HTTP导出器
	// 默认地址：localhost:4318，是OTel Collector / Jaeger OTLP HTTP接收端口
	exporter, err := otlptracehttp.New(context.Background(),
		otlptracehttp.WithEndpoint("localhost:4318"),
		otlptracehttp.WithInsecure(), // 本地测试关闭TLS；生产环境去掉
	)
	if err != nil {
		return nil, fmt.Errorf("创建otlp exporter失败: %w", err)
	}

	// 2. 定义Resource资源信息，标记当前服务
    //`resource`：资源对象，所有从这个服务上报的 Span 都会带上这组属性。在 Jaeger 页面可以按服务名筛选链路。
    //`semconv`是 OTel 语义规范，统一字段名，避免大家自定义 key 造成混乱。
	res, err := resource.New(context.Background(),
		resource.WithAttributes(
			// 服务名称，在Jaeger/Grafana中用来区分不同微服务
			semconv.ServiceNameKey.String("user-service"),
			semconv.ServiceVersionKey.String("v0.0.1"),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("创建resource失败: %w", err)
	}

	// 3. 创建TracerProvider
    //- `WithBatcher(exporter)`：批处理器，后台协程定时批量发送 Span，减少网络请求；
    //- `WithSampler`：采样器：决定一条trace要不要采集
   // - `AlwaysSample()`：测试环境，所有请求都采集 Trace；
   // - `TraceIDRatioBased(0.1)`：生产，10% 采样，高并发减少开销。
    
	tp := sdktrace.NewTracerProvider(
		// 批量上报Span
		sdktrace.WithBatcher(exporter),
		// 绑定服务资源属性
		sdktrace.WithResource(res),
		// 采样策略：AlwaysSample 全部采样，测试环境使用；生产改成概率采样
		sdktrace.WithSampler(sdktrace.AlwaysSample()),
	)
    

	// 4. 设置全局TracerProvider，全局otel.Tracer()会使用这个实例
    //- `otel.SetTracerProvider(tp)`：注册全局 TracerProvider；后续`otel.Tracer()`拿到的 tracer 都由 tp 管理；
  //- `TextMapPropagator`：传播器，负责把 TraceID、父 SpanID 打包成`traceparent`字符串放到 HTTP Header；下游服务读取 Header 还原上下文。
	otel.SetTracerProvider(tp)
	// 设置全局上下文传播器，默认W3C traceparent，自动处理header透传
	otel.SetTextMapPropagator(otel.GetTextMapPropagator())

	return tp, nil
    //`return tp`：返回 tp，main 函数 defer 调用`tp.Shutdown()`。**重要**：程序直接 kill 掉，缓冲区没发送完的 Span 会丢失；Shutdown 会等待剩余 Span 上报完成。
}

// handleOrder 业务处理函数
func handleOrder(w http.ResponseWriter, r *http.Request) {
	// 从http请求r中拿到携带trace信息的context
	ctx := r.Context()

	// 获取Tracer，参数是模块名
	tracer := otel.Tracer("user-service-handler")

	// ==========手动新建子Span：校验订单==========
	// Start：传入父ctx，span名称；返回新ctx（携带新span信息）和span对象
	ctx, spanValidate := tracer.Start(ctx, "Validate Order")
	defer spanValidate.End() // 函数退出结束span，记录耗时！不能忘记defer

	// 给span添加自定义属性
	spanValidate.SetAttributes(semconv.HTTPRouteKey.String("/api/order"))

	// 模拟业务耗时
	time.Sleep(100 * time.Millisecond)

	// ==========手动新建子Span：创建订单DB操作==========
	_, spanDB := tracer.Start(ctx, "Create Order (DB)")
	defer spanDB.End()
	spanDB.SetAttributes(semconv.DBSystemKey.String("mysql"))
	time.Sleep(200 * time.Millisecond)

	// ==========手动新建子Span：发布Kafka事件==========
	_, spanKafka := tracer.Start(ctx, "Publish Event (Kafka)")
	defer spanKafka.End()
	spanKafka.SetAttributes(semconv.MessagingSystemKey.String("kafka"))
	time.Sleep(50 * time.Millisecond)

	w.Write([]byte("order success"))
}

func main() {
	// 初始化追踪
	tp, err := initTracer()
	if err != nil {
		log.Fatalf("init tracer error: %v", err)
	}
	// 程序退出时优雅关闭TracerProvider，刷新缓冲区剩余Span
	defer func() {
		if err := tp.Shutdown(context.Background()); err != nil {
			log.Printf("shutdown tracer provider error: %v", err)
		}
	}()

	mux := http.NewServeMux()
	mux.HandleFunc("/api/order", handleOrder)

	// otelhttp.NewHandler：OTel http中间件
	// 作用：自动创建【根Span】POST /api/order；自动解析请求头traceparent；自动填充http标准属性
	server := &http.Server{
		Addr:    ":8080",
		Handler: otelhttp.NewHandler(mux, "http-server"),
	}

	log.Println("server start at :8080")
	log.Fatal(server.ListenAndServe())
}
```

### 助理解

#### 阶段 1：程序启动（main 函数从上到下，**只执行一次**）

1. 创建 `exporter`（otlptracehttp.New）

2. 创建 `res` resource，定义服务名

3. 创建 `tp := sdktrace.NewTracerProvider(WithBatcher(exporter), WithResource(res), WithSampler(...))`

4. ✅ 

   ```
   otel.SetTracerProvider(tp)
   ```

   > 将我们实例化的 tp 赋值给 OTel 包的全局变量。 后续所有地方调用`otel.Tracer()`，默认从这个全局 tp 获取追踪器。

5. ✅ 

   ```
   otel.SetTextMapPropagator(otel.GetTextMapPropagator())
   ```

   > 将传播器存入全局；otelhttp 中间件请求到达时，读取这个全局 propagator 来解析 traceparent header。

6. 注册路由：

   ```
   otelhttp.NewHandler(业务handler, "POST /api/order")
   ```

   > 👉 NewHandler**此时不会创建 Span**！只是包装 handler，保存引用，**不会立刻使用全局 tp/propagator**。

7. 启动 http 服务 `ListenAndServe`，阻塞等待请求。

> ⚠️重点：**启动阶段只是注册组件，不会产生任何 Span。Span 只有 HTTP 请求到达的时候才会创建。**

#### 阶段 2：HTTP 请求到达（每来一次请求，完整跑一遍）

1. 请求抵达服务，进入 otelhttp 中间件逻辑

2. 中间件读取 HTTP Header `traceparent`

3. 读取全局 TextMapPropagator

   ，调用

   ```
   propagator.Extract()
   ```

   - 解析 traceparent，拿到上游 TraceID、父 SpanID，生成携带追踪信息的新 ctx

4. **读取全局 TracerProvider**，用 tp 创建**根 Span：POST /api/order**

5. 把新 ctx 传给你的业务 `handleOrder(r *http.Request)`

6. `ctx := r.Context()` 在 handler 拿到带追踪信息的上下文

7. ```
   tracer := otel.Tracer("user-service-handler")
   ```

   > 从**全局 TracerProvider**获取 tracer 对象

8. ```
   ctx, spanValidate := tracer.Start(ctx, "Validate Order")
   ```

   > 根据 ctx 里的父 Span（根 Span）创建子 Span

9. 业务执行，sleep 模拟耗时；defer 触发 `spanValidate.End()` 结束子 Span

10. 后续继续创建其他子 Span：Create Order、Publish Event

11. 业务 handler 执行完毕，回到 otelhttp 中间件

12. 中间件自动调用根 Span 的 End ()，记录 http 状态码、耗时

13. Batcher 批量收集 Span，**异步**交给 exporter，发送到 Jaeger

#### 阶段 3：程序退出（触发 defer）

1. `tp.Shutdown(ctx)`：刷新 Batcher 中剩余 Span，关闭 exporter 连接



## 四、Trace 链路示例还原

举例：

```text
TraceID: abc123（整条链路共用）
├─ Span: POST /api/order 【根Span，网关】
    ├─ Span: Validate Order 【订单服务，子Span】
    ├─ Span: Create Order (DB)【订单服务，子Span】
    ├─ Span: Publish Event (Kafka)【订单服务，子Span】
    └─ Span: Process Payment【支付服务，子Span】
        └─ Span: Update Balance【支付服务，孙Span】
```

在 Jaeger/Grafana 里查看这条 Trace：

- 能看到每个 Span 耗时；
- 可以看到调用顺序；
- 如果某一步报错，会标记 Span 状态为 Error，带上异常堆栈。

## 五、补充常见概念

1. **根 Span (Root Span)**：一条 Trace 的第一个 Span，没有父 Span。一般是网关收到外部请求创建；
2. **采样（Sampling）**：高并发服务，全量上报 Trace 开销巨大。可以只采样一部分请求（比如 10%）；
3. OTel Collector：可选中间组件，服务把 Trace 发给 Collector，Collector 再转发给 Jaeger/Zipkin，做统一处理；
4. 可观测性三大支柱：
   - Trace：调用链路、耗时（分布式追踪）
   - Metrics：指标，CPU、QPS、延迟（Prometheus）
   - Logs：日志文本

## 六、常见误区

1. Trace ≠ 日志：Trace 结构化记录调用链路，日志是文本；OTel 可以把日志关联到 Span；
2. Trace 不能解决所有问题：适合排查**调用链超时、慢调用**；单纯本地代码问题，日志 + metrics 更合适；
3. TraceID 只是透传标识，**不会自动传递**，框架埋点（如 otelhttp、otelgrpc）会自动处理 header，裸写代码需要手动透传 context。

## 七、Go 项目上手简要流程

1. 引入 OpenTelemetry 相关 Go SDK 包；
2. 编写`initTracer()`初始化 TracerProvider；
3. 使用 OTel 提供的 http/gRPC 中间件，自动埋点（自动创建 Span，自动透传 traceparent）；
4. 本地启动 Jaeger 接收 Trace；
5. 请求服务，访问 Jaeger UI，查看可视化 Trace 链路。





```go
func main() {
     // 先初始化 tracerProvider（前面写的 initTracer）
	tp, err := initTracer()
	if err != nil {
		log.Fatal(err)
	}
	defer tp.Shutdown(context.Background())

    // 重点：用 otelhttp.NewHandler 包装我们自己的 handleOrder
    // 这一步就是注册追踪中间件！
	http.Handle("/api/order", otelhttp.NewHandler(
		http.HandlerFunc(handleOrder),
		"POST /api/order", // 根Span的名字，就是链路里的 POST /api/order
	))

  	log.Println("server start :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

