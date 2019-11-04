# 入门

在您的本地环境中启动并运行jaeger

##仪表

您的应用程序必须经过检测,然后才能将跟踪数据发送到Jaeger后端。
检查[Client Libraries](./client-library.md)部分,
以获取有关如何使用OpenTracing API以及如何初始化和配置Jaeger跟踪器的信息。

## All in One

All-in-one是用于快速本地测试的可执行文件,它使用内存存储组件启动Jaeger UI, collector, query, agent。
a
启动all-in-one最简单方法是使用发布到DockerHub的预构建映像(单个命令行)。

```
$ docker run -d --name jaeger\
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 9411:9411 \
  jaegertracing/all-in-one:{{<currentVersion>}}
```

或从[而二进制发行历史中](https://www.jaegertracing.io/download/)运行`jaeger-all-in-one(.exe)`可执行文件:

```
$ jaeger-all-in-one --collector.zipkin.http-port=9411
```

然后,您可以导航到`http://localhost:16686`以访问Jaeger UI。

容器暴露以下端口:

Port  | Protocol | Component | Function
----- | -------  | --------- | ---
5775  | UDP      | agent     | accept `zipkin.thrift` over compact thrift protocol (deprecated, used by legacy clients only)
6831  | UDP      | agent     | accept `jaeger.thrift` over compact thrift protocol
6832  | UDP      | agent     | accept `jaeger.thrift` over binary thrift protocol
5778  | HTTP     | agent     | serve configs
16686 | HTTP     | query     | serve frontend
14268 | HTTP     | collector | accept `jaeger.thrift` directly from clients
14250 | HTTP     | collector | accept `model.proto`
9411  | HTTP     | collector | Zipkin compatible endpoint (optional)


## Kubernetes和OpenShift

* Kubernetes模板:[https://github.com/jaegertracing/jaeger-kubernetes](https://github.com/jaegertracing/jaeger-kubernetes)
* Kubernetes operator:[https://github.com/jaegertracing/jaeger-operator](https://github.com/jaegertracing/jaeger-operator)
* OpenShift模板:[https://github.com/jaegertracing/jaeger-openshift](https://github.com/jaegertracing/jaeger-openshift)

## 示例应用程序:HotROD

HotROD(按需乘车)是一个演示应用程序,由多个微服务组成
说明了[OpenTracing API](http://opentracing.io)的用法。
博客文章中提供了教程/演练:
[使用OpenTracing进行HotROD之旅](https://medium.com/@YuriShkuro/take-opentracing-for-a-hotrod-ride-f6e3141f7941)。

它可以独立运行,但需要Jaeger后端才能查看跟踪。

### 特性

- 通过数据驱动的依赖关系发现整个系统的架构图。
- 查看请求时间表和错误；了解应用程序的工作方式。
- 查找延迟和缺乏并发的来源。
- 高度关联的日志记录。
- 使用行李传播以:
  - 诊断请求间争用(排队)。
  - 在服务中花费的属性时间。
- 使用具有OpenTracing集成的开源库来获取
    与供应商无关的工具是免费的。

### 先决条件

- 您需要在计算机上安装Go 1.11或更高版本才能从源代码运行。
- 需要[运行Jaeger后端](＃all-in-one)才能查看跟踪。

### 启动

#### 源代码启动

```
mkdir -p $ GOPATH/src/github.com/jaegertracing
cd $ GOPATH/src/github.com/jaegertracing
git clone git@github.com:jaegertracing/jaeger.git杰格
杰格
进行安装
去运行./examples/hotrod/main.go全部
```
#### docker启动

```
$ docker run --rm -it \
  --link jaeger\
  -p8080-8083:8080-8083 \
  -e JAEGER_AGENT_HOST="jaeger" \
  jaegertracing/example-hotrod:{{<currentVersion>}} \
  all
```

#### 来自二进制分发

从[二进制发行档案](https://www.jaegertracing.io/download/)中运行`example-hotrod(.exe)`可执行文件:

```
$ example-hotrod all
```

然后导航到"http://localhost:8080"


## 从Zipkin迁移

Collector服务公开了Zipkin兼容的REST API`/api/v1/spans`,该API同时接受Thrift和JSON。还有支持JSON和Proto的`/api/v2/spans`。
默认情况下,它是禁用的。可以使用--collector.zipkin.http-port = 9411启用。

Zipkin[Thrift](https://github.com/jaegertracing/jaeger-idl/blob/master/thrift/zipkincore.thrift) IDL和Zipkin[Proto](https://github.com/jaegertracing/jaeger-idl/blob/master/proto/zipkin.proto) IDL文件可以在[jaegertracing/jaeger-idl](https://github.com/jaegertracing/jaeger-idl)存储库中找到。
它们与[openzipkin/zipkin-api](https://github.com/openzipkin/zipkin-api)兼容[Thrift](https://github.com/openzipkin/zipkin-api/blob/master/thrift/zipkinCore.thrift) 和[Proto](https://github.com/openzipkin/zipkin-api/blob/master/zipkin.proto)。
