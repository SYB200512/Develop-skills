# Go 可观测性实战：OpenTelemetry + Prometheus + Jaeger

实现 **Tracing（链路追踪）+ Metrics（指标）+ Logs（日志）** 三者合一，也就是常说的可观测性三板斧。 依赖：

- OpenTelemetry：全链路追踪
- Jaeger：查看 trace 链路
- Prometheus：采集指标
- Zap /otelzap：日志自动注入 traceID 项目结构极简，直接复制运行。



通俗理解一下：

**这段代码在一个 HTTP 服务里，同时埋了追踪 (Trace)、指标 (Metric)、日志 (Log)，也就是可观测性的三大件。**

> 类比理解： 服务 = 一家外卖店 
>
> Trace（Jaeger）= 订单完整流转录像：接单 → 后厨做饭 → 打包，每一步花多久，哪里卡住 
>
> Metrics（Prometheus）= 店铺监控看板：每秒多少单、平均出餐耗时、失败率 
>
> Log（Zap）= 纸质记录本：每一笔订单的详细信息，并且记录本上写着这个订单的录像编号（traceID） `context.Context` = 订单小票，从头到尾跟着这个订单走，小票上就印着 traceID

## 1. go.mod

```go
module otel-demo

go 1.21

require (
	go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.50.0
	go.opentelemetry.io/otel v1.25.0
	go.opentelemetry.io/otel/exporters/jaeger v1.25.0
	go.opentelemetry.io/otel/exporters/prometheus v0.47.0
	go.opentelemetry.io/otel/sdk v1.25.0
	go.opentelemetry.io/otel/sdk/metric v0.47.0
	go.opentelemetry.io/otel/sdk/trace v1.25.0
	go.opentelemetry.io/otel/trace v1.25.0
	github.com/prometheus/client_golang v1.19.0
	go.uber.org/zap v1.26.0
	go.uber.org/zap/zapcore v1.26.0
)
```

## 2. main.go

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/jaeger"
	"go.opentelemetry.io/otel/exporters/prometheus"
	"go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	tracesdk "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
	"go.opentelemetry.io/otel/trace"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"go.uber.org/zap"
)

// 全局 tracer
var tracer trace.Tracer

// prometheus 指标定义
var (
	httpRequestDuration = promauto.NewHistogramVec(
		prometheus.HistogramOpts{
			Name:    "http_request_duration_seconds",
			Help:    "Duration of HTTP requests.",
			Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1},
		},
		[]string{"action"},
	)
)

// 初始化 OpenTelemetry：Trace + Metrics
//创建服务资源
func initOTEL(ctx context.Context) (shutdown func(), err error) {
	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName("go-otel-demo"),
			semconv.ServiceVersion("v1.0.0"),
		),
	)
	if err != nil {
		return nil, err
	}

	// 1. Jaeger Trace Exporter（trace exporter追踪数据搬运工）
exporter, err := otlptracehttp.New(context.Background(),
    otlptracehttp.WithEndpoint("localhost:4318"),
    otlptracehttp.WithInsecure(), // 本地测试关闭TLS；生产去掉，用HTTPS
)
if err != nil {
    return nil, fmt.Errorf("创建otlp exporter失败: %w", err)
}
    
    //newprevider:追踪大总管TP
	tp := tracesdk.NewTracerProvider(
		tracesdk.WithBatcher(traceExp),
		tracesdk.WithResource(res),
	)
	otel.SetTracerProvider(tp)
	tracer = otel.Tracer("demo-service")

	// 2. Prometheus Metric Exporter
	metricExp, err := prometheus.New()
	if err != nil {
		return nil, err
	}
	mp := metric.NewMeterProvider(metric.WithReader(metricExp), metric.WithResource(res))
	otel.SetMeterProvider(mp)

	shutdown = func() {
		ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
		defer cancel()
		_ = tp.Shutdown(ctx)
		_ = mp.Shutdown(ctx)
	}
	return shutdown, nil
}

// 模拟业务：内部子调用，新建子span
func bizLogic(ctx context.Context, userID string) error {
	_, span := tracer.Start(ctx, "bizLogic")
	defer span.End()

	// 模拟数据库耗时
	time.Sleep(100 * time.Millisecond)
	zap.L().Info("execute biz logic", zap.String("user_id", userID))
	return nil
}

// HTTP handler：三层埋点（Trace + Log + Metric）
func handleRequest(w http.ResponseWriter, r *http.Request) {
	req := struct {
		UserID string
		Action string
	}{
		UserID: "u_10086",
		Action: "order_create",
	}

	// ========== 1. 链路追踪 ==========
	ctx, span := tracer.Start(r.Context(), "handleRequest")
	defer span.End()

	// ========== 2. 业务日志（ctx 携带 traceId） ==========
	zap.L().InfoContext(ctx, "processing request",
		zap.String("user_id", req.UserID),
		zap.String("action", req.Action),
	)

	// ========== 3. 性能指标 ==========
	timer := prometheus.NewTimer(httpRequestDuration.WithLabelValues(req.Action))
	defer timer.ObserveDuration()

	// 业务调用，产生子span
	if err := bizLogic(ctx, req.UserID); err != nil {
		zap.L().ErrorContext(ctx, "biz failed", zap.Error(err))
		w.WriteHeader(http.StatusInternalServerError)
		_, _ = w.Write([]byte("fail"))
		return
	}

	_, _ = w.Write([]byte("ok"))
}

func main() {
	ctx := context.Background()

	// 初始化 OTEL
	shutdown, err := initOTEL(ctx)
	if err != nil {
		log.Fatalf("init otel failed: %v", err)
	}
	defer shutdown()

	// 初始化 zap logger
	logger, _ := zap.NewProduction()
	zap.ReplaceGlobals(logger)
	defer logger.Sync()

	// 路由
	mux := http.NewServeMux()
	mux.HandleFunc("/api/req", handleRequest)
	// prometheus metrics 端点
	mux.Handle("/metrics", promhttp.Handler())

	// otelhttp 中间件：自动埋点 http server
	handler := otelhttp.NewHandler(mux, "server")

	fmt.Println("server start :8080")
	_ = http.ListenAndServe(":8080", handler)
}
```

## 3. 本地启动依赖（Docker）

```bash
# 启动 Jaeger
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 14268:14268 \
  jaegertracing/all-in-one:latest
```

- Jaeger UI：http://127.0.0.1:16686
- Prometheus 拉取地址：http://127.0.0.1:8080/metrics

## 4. 测试调用

```bash
curl http://127.0.0.1:8080/api/req
```

多次请求，然后打开 Jaeger UI 查看链路。

## 5. 要点对照

1. **context.Context 传递 Trace**`ctx` 在 `handleRequest` → `bizLogic` 一路透传，`tracer.Start(ctx)` 自动继承父 span，traceID 跟着上下文走。日志 `InfoContext(ctx)` 自动把 trace-id 打进日志。
2. **自动埋点 Middleware**`otelhttp.NewHandler` 是 http server 中间件，**自动创建根 span**，采集 http method、status code、url，不用手写根 span。

> 数据库、redis、gRPC 都有对应的 otel interceptor，自动埋点，避免手写大量重复埋点代码。

1. **不要过度埋点** 示例只埋：

- HTTP 入口（中间件自动）
- 核心业务函数 bizLogic 指标只统计核心接口耗时，循环内细粒度函数不埋，否则产生海量 span，成本爆炸。

## 6. 可观测三件套查看

1. **Trace（Jaeger）**：看到完整调用链 handleRequest → bizLogic，每个 span 耗时、日志标签。
2. **Metrics（Prometheus）**：`http_request_duration_seconds` 直方图，统计接口响应分布。
3. **Logs**：zap 日志自带 `trace_id`、`span_id`，可以通过 traceID 在日志系统检索这条请求完整日志。