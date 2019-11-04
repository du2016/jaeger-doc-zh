# 架构

Jaeger的客户端遵守OpenTracing标准中描述的数据模型。阅读[specification](https://github.com/opentracing/specification/blob/master/specification.md)将有助于您更好地理解本节。

## 术语

让我们首先快速了解[OpenTracing规范](https://github.com/opentracing/specification/blob/master/specification.md)定义的术语。

### Span

span表示Jaeger中的逻辑工作单元,具有操作名称,操作的开始时间和持续时间。span可以嵌套并排序以建立因果关系模型。

![Traces and Spans](./img/spans-traces.png)

### Trace

Trace是通过系统的数据/执行路径,可以看作span的有向无环图

## 组件

Jaeger可以通过all-in-one二进制进行部署,其中所有Jaeger后端组件
可以在单个过程中运行,也可以作为可扩展的分布式系统运行,如下所述。
有两个主要的部署选项:

  1. 收集器正在直接写入存储。

  2. 收集者正在写信给Kafka作为初步缓冲区。

![架构](./img/architecture-v1.png)

直接存储架构的图解

![Architecture](./img/architecture-v2.png)
以Kafka作为中间缓冲区的架构插图

本节详细介绍Jaeger的组成部分以及它们之间的关系。它由您的应用程序与之交互的顺序安排。

### Jaeger客户端库

Jaeger客户端是[OpenTracing API](http://opentracing.io)的特定于语言的实现。
它们可用于手动或通过与OpenTracing集成的各种现有开源框架(例如Flask,Dropwizard,gRPC等)来检测应用程序以进行分布式跟踪。

接收服务的服务会在接收到新请求时创建spans,并将上下文信息(spans,id,span id和baggage)附加到传出的请求。
只有ID和id和baggage随请求一起传播；不会传播构成spans的所有其他信息,例如操作名称,日志等。取而代之的是,采样的spans会在后台异步传输到Jaeger Agents的进程外。

该架构的开销很小,并且设计为始终在生产中启用。

请注意,虽然生成了所有trace,但仅采样了一些。对跟踪进行采样将跟踪标记为进一步处理和存储。
默认情况下,Jaeger客户端对0.1％的迹线进行采样(每1000个中的1个),并且能够从代理中检索采样策略。

![上下文传播说明](./img/context-prop.png)
上下文传播的插图
### Agent

Jaeger代理是一个网络守护程序,它侦听通过UDP发送的跨度,然后将其分批发送给收集器。它旨在作为基础结构组件部署到所有主机。
该代理将收集器的路由和发现抽象到远离客户端的位置。

### Collector

Jaeger收集器从Jaeger代理接收跟踪,并通过处理管道运行它们。当前,我们的管道会验证跟踪,为其建立索引,
执行任何转换并最终存储它们.Jaeger的存储设备是可插拔组件,目前支持Cassandra,Elasticsearch和Kafka。

### Query

查询是一项从存储中检索跟踪并托管UI来显示跟踪的服务。

### Ingester

Ingester是一项从Kafka主题读取并写入另一个存储后端(Cassandra,Elasticsearch)的服务。
