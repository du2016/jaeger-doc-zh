# 客户端库

所有Jaeger客户端库都支持[OpenTracing API](http://opentracing.io)。以下资源提供了有关使用OpenTracing来检测应用程序的更多信息:

\*[OpenTracing教程](https://github.com/yurishkuro/opentracing-tutorial)适用于Java,Go,Python,Node.js和C＃

* 深入的博​​客文章[在Go中追踪HTTP请求延迟](https://medium.com/@YuriShkuro/tracing-http-request-latency-in-go-with-opentracing-7cc1282a100a)
* 官方OpenTracing文档和其他资料,请访问[opentracing.io](http://opentracing.io)

  GitHub上的[`opentracing-contrib` org](https://github.com/opentracing-contrib)包含许多存储库,这些存储库具有适用于许多流行框架的现成工具,包括JAXRS和Dropwizard\(Java\),Flask和Django\(Python\),Go std库等

该页面的其余部分包含有关在已经使用OpenTracing API进行了测试的应用程序中配置和实例化Jaeger跟踪器的信息。

## 术语

在本文档中,我们可以互换使用术语client library, instrumentation library, 和 tracer interchangeably。

## 支持的库

正式支持以下客户端库:

| Language | GitHub Repo |
| :--- | :--- |
| Go | jaegertracing/jaeger-client-go |
| Java | jaegertracing/jaeger-client-java |
| Node.js | jaegertracing/jaeger-client-node |
| Python | jaegertracing/jaeger-client-python |
| C++ | jaegertracing/jaeger-client-cpp |
| C\# | jaegertracing/jaeger-client-csharp |

其他语言的图书馆目前正在开发中,请参见[issue＃366](https://github.com/jaegertracing/jaeger/issues/366)。

## 初始化Jaeger Tracer

每种语言的初始化语法略有不同,请参阅相应存储库中的自述文件。 通用模式是不显式创建Tracer,而是使用Configuration类来执行此操作。配置允许 跟踪程序的简单参数化,例如更改默认采样器或Jaeger代理的位置。

## Tracer内部

### 采样

请参阅[此处](https://github.com/du2016/jaeger-doc-zh/tree/2888ea563a0fd2ef7bf79c0d807f3ecd4d1ee924/sampling＃client-sampling-configuration/README.md)。

### Reporters

Jaeger追踪器使用**reporters**来处理完成的spans。通常,Jaeger libraries附带以下reporters:

* **NullReporter**对范围不执行任何操作。它在单元测试中很有用。
* **LoggingReporter**仅记录跨度已完成的事实,通常是通过打印跟踪和跨度ID以及操作名称来记录。
* **CompositeReporter**获取其他报告者列表,并逐个调用它们。
* **RemoteReporter**\(默认\)在内存中缓冲一定数量的完成跨度,并使用发件人\*\*将一批跨度跨进程提交到Jaeger后端。发送方负责将跨度序列化为有线格式\(例如Thrift或JSON\),并与后端组件进行通信\(例如,通过UDP或HTTP\)。

#### EMSGSIZE和UDP缓冲区限制

默认情况下,Jaeger库使用UDP发送器将已完成的spans报告给`jaeger-agent`守护程序。 默认的最大数据包大小为65,000字节,在以下情况下无需分段即可传输 通过环回接口连接到代理。但是,某些操作系统\(尤其是MacOS\)会限制 UDP数据包的最大缓冲区大小,在[此GitHub问题](https://github.com/uber/jaeger-client-node/issues/124)中提出。 如果遇到`EMSGSIZE`错误的问题,请考虑提高内核中的限制\(有关示例,请参见问题\)。 您还可以将客户端库配置为使用较小的最大数据包大小,但这可能会导致 如果您的跨度较大,例如如果您记录大量数据。跨度超过最大数据包大小 被客户端丢弃\(发出度量以表明这一点\)。另一种选择是 使用非UDP传输,例如[Java中的HttpSender](https://github.com/jaegertracing/jaeger-client-java/blob/master/jaeger-thrift/src/main/java/io/jaegertracing/thrift/internal/senders/HttpSender.java)\(目前并非所有语言都可用\)。

### 指标

Jaeger跟踪器会发出各种度量标准,以说明它们开始和完成了多少个跨度或跟踪, 已采样或未采样的跨度或跟踪,解码从入站请求或报告spans到后端的跟踪上下文时是否存在任何错误。

TODO标准化并描述了度量标准名称和标签\(问题[＃572](https://github.com/jaegertracing/jaeger/issues/572),[＃611](https://github.com/jaegertracing/jaeger/问题/611)\)。

## 传播格式

当将`SpanContext`作为对另一服务的请求的一部分在网络上进行编码时, Jaeger客户端库默认为以下指定的编码。 将来,Jaeger将支持即将推出的[W3C跟踪上下文规范](https://github.com/w3c/distributed-tracing)。

### Trace/Span标识

#### 键

`uber-trace-id`

* HTTP不区分大小写
* 在保留标题大小写的协议中为小写

#### 值

`{trace-id}:{span-id}:{parent-span-id}:{flags}`

\*`{trace-id}`

* base16格式的64位或128位随机数

  \*可以是可变长度,较短的值在左侧填充0

  \*某些语言的客户端支持128位迁移中

* 0值无效

  \*`{span-id}`

* base16格式的64位随机数

  \*`{parent-span-id}`

* base16格式的64位值,代表父范围ID

  \*已弃用,

大多数Jaeger客户端会在接收方忽略,但仍将其包含在发送方

* 0值有效,表示"根跨度"\(不忽略时\)

  \*`{flags}`

  \*一字节位图,为两个十六进制数字

  \*位1\(最右,最低有效\)是"采样"标志

  * 1表示跟踪已采样,建议所有下游服务都遵守
  * 0表示不对跟踪进行采样,建议所有下游服务尊重

      \*我们正在考虑一项新功能,如果下游服务发现其跟踪级别太低,则可以对其进行升采样

    \*位2是"调试"标志

    \*调试标志仅应在设置采样标志时设置

    \*指示后端尽力不掉线

    \*其他位未使用

### Baggage

* 键:`uberctx-{baggage-key}`
* 值:URL编码的字符串
* 限制:由于HTTP标头不保留大小写,因此Jaeger建议将行李密钥为小写,例如kebab-case,

  例如我的行李钥匙-1。

