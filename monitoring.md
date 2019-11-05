# 监控

Jaeger本身是一个基于分布式,微服务的系统。 如果您在生产环境中运行它,则可能需要为不同的组件设置足够的监控,例如以确保后端不会被过多的跟踪数据饱和。

## 指标

默认情况下,Jaeger微服务以Prometheus格式公开指标。它由以下命令行选项控制:

* `--metrics-backend`控制如何显示测量结果。默认值为prometheus,另一个选项为expvar,它是公开流程级别统计信息的Go标准机制。
* `--metrics-http-route`指定用于刮擦指标的HTTP端点的名称\(默认情况下为/metrics\)。

每个Jaeger组件都会在它们已经提供的HTTP端口之一上公开度量标准刮取端点:

| 组件 | 端口 |
| :--- | :--- |
| **jaeger-agent** | 14271 |
| **jaeger-collector** | 14269 |
| **jaeger-query** | 16687 |
| **jaeger-ingester** | 14270 |
| _all-in-one_ | 14269 |

## 记录日志

Jaeger组件仅使用结构化日志记录库[go.uber.org/zap](https://github.com/uber-go/zap)配置 为以JSON编码的字符串写入日志行的方式登录到标准,例如:

```text
json
{"level":"info","ts":1517621222.261759,"caller":"healthcheck/handler.go:99","msg":"Health Check server started","http-port":14269,"status":"unavailable"}
```

日志级别可以通过`--log-level`命令行开关进行调整；默认级别为`info`。

