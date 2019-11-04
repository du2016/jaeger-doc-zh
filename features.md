Jaeger用于监视和诊断基于微服务的分布式系统,包括:

* 分布式上下文传播
* 分布式事务监控
* 根本原因分析
* 服务依赖性分析
* 性能/延迟优化

## 高可扩展性

Jaeger后端的设计没有单点故障,可以根据业务需求进行扩展。
例如,Uber上任何给定安装的Jaeger通常每天要处理数十亿个spans。

## 对OpenTracing的原生支持

Jaeger后端,Web UI和工具库已完全设计为支持OpenTracing标准。

* 通过[span reference](https://github.com/opentracing/specification/blob/master/specification.md#references-between-spans)将traces表示为有向无环图
* 支持强类型span标签和结构化日志
* 通过`baggage`支持通用的分布式上下文传播机制

## 多个存储后端

Jaeger支持两个流行的开源NoSQL数据库作为跟踪存储后端: Cassandra 3.4+和Elasticsearch 5.x/6.x/7.x。
正在进行使用其他数据库的社区实验,例如ScyllaDB,InfluxDB,Amazon DynamoDB。Jaeger也提供
带有用于测试设置的简单内存存储。

## 现代Web UI

Jaeger Web UI是使用流行的开源框架(如React)以Javascript实现的。
v1.0中发布了一些性能改进,以允许UI有效处理大量数据,并显示成千上万跨度的跟踪

## 云原生部署

Jaeger后端作为Docker 镜像的集合进行分发。二进制文件支持各种配置方法,
包括命令行选项,环境变量和多种格式(yaml,toml等)的配置文件。
[Kubernetes operator](https://github.com/jaegertracing/jaeger-operator),[Kubernetes模板](https://github.com/jaegertracing/jaeger-kubernetes)协助部署到Kubernetes集群。
和[helm chat](https://github.com/kubernetes/charts/tree/master/incubator/jaeger)。

## 可观察性

默认情况下,所有Jaeger后端组件都公开[Prometheus](https://prometheus.io/)指标(还支持其他指标后端)。
使用结构化日志库[zap](https://github.com/uber-go/zap)将日志写到标准输出。

## 与Zipkin的向后兼容性

尽管我们建议使用OpenTracing API来对应用程序进行检测并绑定到Jaeger客户端库,以便从其他地方无法获得的高级功能中受益
如果您的组织已经使用Zipkin库对工具进行了投资,
则不必重写所有代码.Jaeger通过在HTTP上接受Zipkin格式(Thrift,JSON v1/v2和Protobuf)的spans来提供与Zipkin的向后兼容性。
从Zipkin后端切换只是将流量从Zipkin库路由到Jaeger后端的问题
