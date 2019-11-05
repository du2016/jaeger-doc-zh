# 客户端功能

下表提供了现有客户端库的功能矩阵.标有"？"的单元格表示不知道给定的客户端是否支持给定的功能,是否需要其他研究和文档更新。

## 用于报告的数据格式和传输范围跨到Jaeger后端:

| Feature | Go | Java | Node.js | Python | C++ | C\# |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Report jaeger.thrift over UDP | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Report jaeger.thrift over HTTP | x | ✓ | x | x | ? | ✓ |
| Report Zipkin Thrift over HTTP | ✓ | x | x | x | x | x |

## 进程间传播格式\(头\)

| Feature | Go | Java | Node.js | Python | C++ | C\# |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Uber’s original headers | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Zipkin’s B3 headers | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| W3C Trace Context headers | coming | coming | coming | coming | coming | coming |
| Support inbound jaeger-debug-id header | when there is no trace | when there is no trace | ❔ | ❔ | ❔ | when there is no trace |
| Accept baggage from jaeger-baggage headers | when there is no trace | ❌ | ❔ | ❔ | ❔ | ❌ |
| Support for 128bit Trace ID | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

## 指标

| Feature | Go | Java | Node.js | Python | C++ | C\# |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Support standard tracer metrics \(number of spans started, finished, reported, etc.\) | ✅ | ✅ | ✅ | ✅ | ❔ | ✅ |
| Support standard RPC metrics | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Metrics in Prometheus format | ✅ | ✅ | ❌ | ✅ | ❔ | ❔ |

## Tracer配置

| Feature | Go | Java | Node.js | Python | C++ | C\# |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Support declarative tracer configuration | ✅ | ✅ | ✅ | ✅ | ❔ | ✅ |
| Allow configuring tracer tags, aka process tags | ✅ | ✅ | ✅ | ✅ | ❔ | ✅ |
| Allow remote configuration of samplers | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Remotely configurable adaptive sampler | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Remotely configurable baggage restrictions | ✅ | coming | coming | coming | ❌ | coming |

## 通过环境变量配置Tracer

| Feature | Go | Java | Node.js | Python | C++ | C\# |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| JAEGER\_SERVICE\_NAME defines service name that will be associated with the emitted spans. | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| JAEGER\_TAGS defines a comma-separated list of static tags, aka “tracer tags”, e.g. hostname=foobar,my.app.version=1.2.3. These tags are added as the Process tags to each span. | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| JAEGER\_DISABLED \(not recommended\) when set to true, instructs the Configuration to return a no-op tracer. Neither trace/span IDs nor baggage will be propagated with a no-op tracer. Instead of disabling the tracer completely, use const sampler with 0 param, which will minimize the overhead but keep the propagation going. | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| JAEGER\_AGENT\_HOST defines hostname for reporting spans over UDP/Thrift. To avoid packet loss, the agent is expected to run on the same machine as the application. This var is useful when there are multiple networking namespaces on the host. | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| JAEGER\_AGENT\_PORT defines port for reporting spans over UDP/Thrift. | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| JAEGER\_ENDPOINT defines the URL for reporting spans via HTTP/Thrift. This setting allows for a deployment mode where spans are submitted directly to the collector. | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| JAEGER\_AUTH\_TOKEN defines an optional auth token when reporting over HTTP. | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| JAEGER\_USER can be used for basic authentication when reporting over HTTP. | ✅ | ✅ | ✅    ❌ | ❌ | ✅ |  |
| JAEGER\_PASSWORD can be used for basic authentication when using reporting over HTTP. | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| JAEGER\_REPORTER\_LOG\_SPANS instructs the Reporter to log finished span IDs. The reporter may need to be given a Logger for this option to take effect. | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| JAEGER\_REPORTER\_MAX\_QUEUE\_SIZE defines the max size of the in-memory buffer used to keep spans before they are sent out. | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| JAEGER\_REPORTER\_FLUSH\_INTERVAL defines how frequently the report flushes span batches. Reporter can also flush the batch if the batch size reaches the maximum UDP packet size \(~64Kb\). | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| JAEGER\_SAMPLER\_TYPE defines the type of sampler to use, e.g. probabilistic, or const \(see Sampling\). | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| JAEGER\_SAMPLER\_PARAM provides configuration value to the sampler, e.g. probability=0.001 \(see Sampling\). | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| JAEGER\_SAMPLER\_MANAGER\_HOST\_PORT defines the address of HTTP server that services client configuration, such as sampling strategies, baggage restrictions, throttling config, etc. The variable name is a legacy misnomer from the time when the server only provided the sampling strategies. At the moment only jaeger-agent implements this REST API. | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| JAEGER\_SAMPLER\_REFRESH\_INTERVAL defines how often the sampler polls the config server for updates to the samling strategies. | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| JAEGER\_SAMPLER\_MAX\_OPERATIONS instructs the adaptive sampler to limit how many distinct operation names the sampler will track, to avoid unbound memory usage. | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| JAEGER\_PROPAGATION defines the propagation format used by the tracer. Supported values are jaeger \(defined here\) and b3 \(defined here\). | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ |
| JAEGER\_TRACEID\_128BIT if true, instructs the tracer to generate 128bit trace IDs instead of the default 64bit. In the future 128bit will become the default. | ❌ | ✅ | ❌ | ✅ | ❌ | ✅ |
| JAEGER\_RPC\_METRICS when true, enables additional generation of RPC metrics from the tracing instrumentation. This is an experimental feature in the Go client. See also [https://github.com/opentracing-contrib/java-metrics](https://github.com/opentracing-contrib/java-metrics). | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

