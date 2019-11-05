# 故障排查

解决常见问题 Jaeger由不同的组件组成,每个组件都可能在其自己的主机中运行。这些运动部件之一可能无法正常工作,从而导致跨度无法处理和存储。 如果出现问题,请确保检查此处列出的项目。

## 验证抽样策略

在进行其他所有操作之前,请确保确认正在使用哪种采样策略。默认情况下,Jaeger使用概率抽样策略,将有1/1000的机会报告跨度。 但是,出于开发目的或用于低流量场景,应该对每个跟踪进行采样。

通常,可以通过环境变量" JAEGER_SAMPLER\_TYPE"和" JAEGER\_SAMPLER\_PARAM"设置采样策略,但是有关所使用的采样策略的更多详细信息, 请参考Jaeger客户文档中所使用的语言。使用Jaeger \_Java_ Client时,通常在创建跟踪器时通过检测应用程序提供的日志记录功能将策略打印出来:

```text
2018-12-10 16:41:25 INFO  Configuration:236 - Initialized  tracer=JaegerTracer(..., sampler=ConstSampler(decision=true,  tags={sampler.type=const, sampler.param=true}), ...)
```

在诊断为什么其他组件未接收到跨度时,请确保将客户端配置为对每个跟踪采样,将环境变量JAEGER\_SAMPLER\_TYPE设置为const并将JAEGER\_SAMPLER\_PARAM设置为1。

## 使用日志记录报告程序

大多数Jaeger客户端都能够记录正在报告到由检测应用程序提供的记录工具的跨度。通常,这可以通过将环境变量" JAEGER_REPORTER\_LOG\_SPANS"设置为" true"来完成,但是请参阅Jaeger客户的文档以获取所使用的语言。在某些语言中,尤其是在Go和Node.js中,没有事实上的标准日志记录工具,因此您需要将记录器显式传递给客户端,该客户端实现由Jaeger Client定义的非常狭窄的Logger接口。使用Jaeger \_Java_ Client时,跨度报告如下:

```text
2018-12-10 17:20:54 INFO  LoggingReporter:43 - Span reported:  e66dc77b8a1e813b:6b39b9c18f8ef082:a56f41e38ca449a4:1 -  getAccountFromCache
```

上面的日志条目包含三个ID:跟踪ID" e66dc77b8a1e813b",跨度的ID" 6b39b9c18f8ef082"和跨度的父ID" a56f41e38ca449a4"。当后端组件的日志级别设置为\`debug'时,跨度和跟踪ID应该在其标准输出上可见\(请参阅"在后端组件中增加日志记录"下的更多信息\)。

记录报告程序遵循采样器做出的采样决策,这意味着如果记录了跨度,则跨度也应到达代理或收集器。

## 绕过Jaeger代理

默认情况下,Jaeger客户端被配置为通过UDP将跨度发送到在" localhost"上运行的Jaeger代理。由于某些网络设置可能会丢弃或阻止UDP数据包或施加大小限制,因此可以将Jaeger客户端配置为绕过代理,将跨度直接发送到收集器。某些客户端\(例如Jaeger _Java_ Client\)支持环境变量" JAEGER_ENDPOINT"\(可用于指定收集器的位置\),例如_[http://jaeger-collector:14268/api/traces。有关所使用的语言,请参阅Jaeger客户的文档。在将](http://jaeger-collector:14268/api/traces。有关所使用的语言,请参阅Jaeger客户的文档。在将)_" JAEGER\_ENDPOINT"属性配置为收集器的端点后,Jaeger \_Java_ Client在创建跟踪器时会记录以下内容:

```text
2018-12-10 17:06:30 INFO  Configuration:236 - Initialized  tracer=JaegerTracer(...,  reporter=CompositeReporter(reporters=[RemoteReporter(sender=HttpSender(),  ...), ...]), ...)
```

> 当无法建立与Jaeger Collector的连接时,Jaeger Java Client将不会失败。跨度将被收集并放置在内部缓冲区中。一旦建立连接,它们最终可能到达收集器,或者在缓冲区达到其最大大小时被丢弃。

## 增加后端组件中的日志记录

当日志级别设置为"调试"时,Jaeger代理和收集器会提供有用的调试信息。记录代理收到的每个UDP数据包,以及代理发送给收集器的每个批次。收集器还记录接收到的每个批次,并记录存储在永久存储器中的每个跨度。

使用–log-level = debug标志启动Jaeger代理时,会发生以下情况:

```text
{"level":"debug","ts":1544458854.5367086,"caller":"processors/thrift_processor.go:113","msg":"Span(s) received by the agent","bytes-received":359}
{"level":"debug","ts":1544458854.5408711,"caller":"tchannel/reporter.go:133","msg":"Span batch submitted by the agent","span-count":3}
```

在收集器端,当标记\`--log-lev时,这些是预期的日志条目。

指定el = debug\`:

```text
{"level":"debug","ts":1544458854.5406284,"caller":"app/span_handler.go:90","msg":"Span batch processed by the collector.","ok":true}
{"level":"debug","ts":1544458854.5406587,"caller":"app/span_processor.go:105","msg":"Span written to the storage by the collector","trace-id":"e66dc77b8a1e813b","span-id":"6b39b9c18f8ef082"}
{"level":"debug","ts":1544458854.54068,"caller":"app/span_processor.go:105","msg":"Span written to the storage by the collector","trace-id":"e66dc77b8a1e813b","span-id":"d92976b6055e6779"}
{"level":"debug","ts":1544458854.5406942,"caller":"app/span_processor.go:105","msg":"Span written to the storage by the collector","trace-id":"e66dc77b8a1e813b","span-id":"a56f41e38ca449a4"}
```

## 检查/metrics端点

对于不可能或不希望在收集器端增加日志记录的情况,可以使用/metrics端点检查是否收到了特定服务的跨度。假设Jaeger Collector可以在名为" jaeger-collector"的主机下使用,下面是一个" curl"调用示例,用于获取指标:

```text
curl http://jaeger-collector:14269/metrics
```

以下指标特别重要:

```text
jaeger_collector_spans_received
jaeger_collector_spans_saved_by_svc
jaeger_collector_traces_received
jaeger_collector_traces_saved_by_svc
```

对于同一服务,前两个指标应具有相似的值。同样,两个"跟踪"指标也应具有相似的值。例如,这是按预期运行的安装程序的示例:

```text
jaeger_collector_spans_received{debug="false",format="jaeger",svc="order"} 8
jaeger_collector_spans_saved_by_svc{debug="false",result="ok",svc="order"} 8
jaeger_collector_traces_received{debug="false",format="jaeger",svc="order"} 1
jaeger_collector_traces_saved_by_svc{debug="false",result="ok",svc="order"} 1
```

