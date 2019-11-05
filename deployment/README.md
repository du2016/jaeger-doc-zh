# 部署

Jaeger主要后端组件在Docker Hub上作为Docker映像发布:

| Component | Repository |
| :--- | :--- |
| **jaeger-agent** | [hub.docker.com/r/jaegertracing/jaeger-agent/](https://hub.docker.com/r/jaegertracing/jaeger-agent/) |
| **jaeger-collector** | [hub.docker.com/r/jaegertracing/jaeger-collector/](https://hub.docker.com/r/jaegertracing/jaeger-collector/) |
| **jaeger-query** | [hub.docker.com/r/jaegertracing/jaeger-query/](https://hub.docker.com/r/jaegertracing/jaeger-query/) |
| **jaeger-ingester** | [hub.docker.com/r/jaegertracing/jaeger-ingester/](https://hub.docker.com/r/jaegertracing/jaeger-ingester/) |

有用于通过以下方式运行Jaeger的编排模板:

* Kubernetes:[github.com/jaegertracing/jaeger-kubernetes](https://github.com/jaegertracing/jaeger-kubernetes),
* OpenShift:[github.com/jaegertracing/jaeger-openshift](https://github.com/jaegertracing/jaeger-openshift)。

## 配置选项

Jaeger二进制文件可以通过多种方式进行配置\(按优先级从高到低的顺序\):

* 命令行参数,
* 环境变量,
* JSON,TOML,YAML,HCL或Java属性格式的配置文件。

要查看选项的完整列表,请使用`help`命令运行二进制文件。 仅当选择了存储类型时,才会列出特定于特定存储后端的选项。 例如,要查看具有Cassandra存储的收集器中的所有可用选项:

```bash
$ docker run --rm \
    -e SPAN_STORAGE_TYPE=cassandra \
    jaegertracing/jaeger-collector:1.14 \
    help
```

为了通过环境变量提供配置参数,请找到相应的命令行选项并将其名称转换为UPPER\_SNAKE\_CASE,例如:

| Command line option | Environment variable |
| :--- | :--- |
| `--cassandra.connections-per-host` | `CASSANDRA_CONNECTIONS_PER_HOST` |
| `--metrics-backend` | `METRICS_BACKEND` |

## 客户端

Jaeger客户端库希望**jaeger-agent**进程在每个主机上本地运行。 代理公开以下端口:

| Port | Protocol | Function |
| :--- | :--- | :--- |
| 6831 | UDP | accept\[jaeger.thrift\]\[jaeger-thrift\] in `compact` Thrift protocol used by most current Jaeger clients |
| 6832 | UDP | accept\[jaeger.thrift\]\[jaeger-thrift\] in `binary` Thrift protocol used by Node.js Jaeger client \(because\[thriftrw\]\[thriftrw\] npm package does not support `compact` protocol\) |
| 5778 | HTTP | serve configs, sampling strategies |
| 5775 | UDP | accept\[zipkin.thrift\]\[zipkin-thrift\] in `compact` Thrift protocol \(deprecated; only used by very old Jaeger clients, circa 2016\) |
| 14271 | HTTP | Healthcheck at `/` and metrics at `/metrics` |

它可以直接在主机上或通过Docker执行,如下所示:

```bash
##确保仅公开您在部署方案中使用的端口!
docker run \
  --rm \
  -p6831:6831/udp \
  -p6832:6832/udp \
  -p5778:5778/tcp \
  -p5775:5775/udp \
  jaegertracing/jaeger-agent:{{<currentVersion>}}
```

### 发现系统集成

代理可以点对点连接到单个收集器地址,该地址可以是 跨多个收集器由另一个基础结构组件\(例如DNS\)进行负载平衡。 还可以为代理配置一个静态的收集器地址列表。

在Docker上,可以使用如下命令:

```bash
docker run \
  --rm \
  -p5775:5775/udp \
  -p6831:6831/udp \
  -p6832:6832/udp \
  -p5778:5778/tcp \
  jaegertracing/jaeger-agent:{{<currentVersion>}} \
  --reporter.grpc.host-port = jaeger-collector.jaeger-infra.svc:14250
```

或使用`--reporter.tchannel.host-port = jaeger-collector.jaeger-infra.svc:14267`来使用 传统的tchannel reporter。

使用gRPC时,可以使用几种选项来进行负载均衡和域名解析:

* 单连接,无负载均衡。如果指定单个`host:port`,这是默认设置.\(例如:`--reporter.grpc.host-port = jaeger-collector.jaeger-infra.svc:14250`\)
* 主机名和循环负载平衡的静态列表。这是用逗号分隔的地址列表得到的.\(例如:`reporter.grpc.host-port = jaeger-collector1:14250,jaeger-collector2:14250,jaeger-collector3:14250`\)
* 动态DNS解析和循环负载平衡。要获得此行为,请在地址前加上`dns:///`,

  gRPC会尝试使用SRV记录来解析主机名\(用于[外部负载均衡](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)\)、

  TXT记录\(用于[服务配置](https://github.com/grpc/grpc/blob/master/doc/service_config.md)\)和A记录。

  请参阅[gRPC名称解析文档](https://github.com/grpc/grpc/blob/master/doc/naming.md)和

  [dns\_resolver.go实现](https://github.com/grpc/grpc-go/blob/master/resolver/dns/dns_resolver.go)获取更多信息

  \(例子: `--reporter.grpc.host-port=dns:///jaeger-collector.jaeger-infra.svc:14250`\)

### 客户端级别标签

Jaeger支持代理级标签,可以将其添加到通过客户端的所有span的过程标签中。 通过命令行标志`--jaeger.tags = key = value`支持。 标签也可以通过这样的环境标志来设置`--jaeger-tags=key=${envFlag:defaultValue}`- 标签值将被设置为`envFlag`环境键的值如果没有设置将被设置为和`defaultValue`, tchannel reporter不支持此功能,使用标志`--collector.host-port`或`--reporter.tchannel.host-port`启用。

## Collectors

Collectors是无状态的,因此可以并行运行多个**jaeger-collector**实例。 收集器几乎不需要任何配置,除了Cassandra群集的位置, 通过`--cassandra.keyspace`和`--cassandra.servers`选项,或Elasticsearch集群的位置,通过 `--es.server-urls`,取决于指定的存储。查看所有命令行选项运行

```bash
go run ./cmd/collector/main.go -h
```

或者,如果您没有源代码

```bash
docker run -it --rm jaegertracing/jaeger-collector:{{<currentVersion>}} -h
```

在默认设置下,collector公开以下端口:

| Port | Protocol | Function |
| :--- | :--- | :--- |
| 14267 | TChannel | used by **jaeger-agent** to send spans in jaeger.thrift format |
| 14250 | gRPC | used by **jaeger-agent** to send spans in model.proto format |
| 14268 | HTTP | can accept spans directly from clients in jaeger.thrift format over binary thrift protocol |
| 9411 | HTTP | can accept Zipkin spans in Thrift, JSON and Proto \(disabled by default\) |
| 14269 | HTTP | Healthcheck at `/` and metrics at `/metrics` |

## 存储后端

Collectors需要持久的存储后端.Cassandra和Elasticsearch是主要支持的存储后端。 其他后端在[此处讨论](https://github.com/jaegertracing/jaeger/issues/638)。

可以通过SPAN\_STORAGE\_TYPE环境变量来传递存储类型。 有效值为cassandra,elasticsearch,kafka\(仅作为缓冲区\),grpc-plugin,badger\(仅适用于all-in-one\)和memory\(仅适用于all-in-one\)。

从1.6.0版开始,可以通过向SPAN\_STORAGE\_TYPE环境变量提供一个以逗号分隔的有效类型列表来同时使用多种存储类型。 重要的是要注意,所有列出的存储类型都用于写入,但是列表中仅第一个类型将用于读取和归档。

### 内存

内存中存储不适用于生产工作负载。它旨在作为一种快速入门的简单解决方案, 该过程结束后,数据将丢失。

默认情况下,存储在内存中的跟踪数量没有限制,但是可以通过`--memory.max-traces`传递一个整型来建立限制。

### Badger-本地存储

自Jaeger 1.9起进行实验

[Badger](https://github.com/dgraph-io/badger)是嵌入式本地存储,仅可用 具有`all-in-one`分布。默认情况下,它用作使用临时文件系统的临时存储。 这可以通过使用`--badger.ephemeral = false`选项来覆盖。

```bash
docker run \
  -e SPAN_STORAGE_TYPE=badger \
  -e BADGER_EPHEMERAL=false \
  -e BADGER_DIRECTORY_VALUE=/badger/data \
  -e BADGER_DIRECTORY_KEY=/badger/key \
  -v <storage_dir_on_host>:/badger \
  -p 16686:16686 \
  jaegertracing/all-in-one:1.14
```

### Cassandra

支持的版本:3.4+

部署Cassandra本身超出了我们的文档范围。一个好 文档的来源是[Apache Cassandra Docs](https://cassandra.apache.org/doc/latest/)。

#### 配置

**最小**

```bash
docker run \
  -e SPAN_STORAGE_TYPE=cassandra \
  -e CASSANDRA_SERVERS=<...> \
  jaegertracing/jaeger-collector:1.14
```

**所有选项**

要查看配置选项的完整列表,可以运行以下命令:

```bash
docker run \
  -e SPAN_STORAGE_TYPE=cassandra  \
  jaegertracing/jaeger-collector:1.14 \
  --help
```

#### 结构脚本

提供了一个脚本来初始化Cassandra键空间和表结构 使用Cassandra的交互式shell `cqlsh`:

```bash
MODE=test sh ./plugin/storage/cassandra/schema/create.sh | cqlsh
```

对于生产部署,请将`MODE = prod DATACENTER = {datacenter}`参数传递给脚本, 其中`{datacenter}`是Cassandra配置/网络拓扑中使用的名称。

该脚本还允许覆盖TTL,键空间名称,复制因子等。 运行不带参数的脚本以查看可识别参数的完整列表。

#### TLS支持

只要已配置,Jaeger就支持TLS客户端到节点的连接 您的Cassandra群集正确。经过验证`cqlsh`后\`,你可以 像这样配置收集器和查询:

```bash
docker run \
  -e CASSANDRA_SERVERS=<...> \
  -e CASSANDRA_TLS=true \
  -e CASSANDRA_TLS_SERVER_NAME="CN-in-certificate" \
  -e CASSANDRA_TLS_KEY=<path to client key file> \
  -e CASSANDRA_TLS_CERT=<path to client cert file> \
  -e CASSANDRA_TLS_CA=<path to your CA cert file> \
  jaegertracing/jaeger-collector:1.14
```

结构工具还支持TLS。您需要制作一个自定义的cqlshrc文件,例如:

```text
＃在需要客户端TLS证书的Cassandra集群中创建架构。
＃
＃为架构docker容器创建一个包含四个文件的卷:
＃cqlshrc:此文件
＃ca-cert:密钥的证书颁发机构
＃client-key:客户端的密钥文件
＃client-cert:与客户端密钥匹配的证书文件
＃
＃如果存在任何类型的DNS不匹配,而您想忽略服务器验证
＃个问题,然后在下面取消注释validate = false。
＃
＃运行容器时,将此卷映射到/root/.cassandra并设置
＃环境变量CQLSH_SSL =-ssl
[ssl]
certfile = ~/.cassandra/ca-cert
userkey = ~/.cassandra/client-key
usercert = ~/.cassandra/client-cert
# validate = false
```

### Elasticsearch

从0.6.0开始在Jaeger中受支持 支持的版本:5.x,6.x,7.x

Elasticsearch不需要初始化 [安装和运行Elasticsearch](https://www.elastic.co/downloads/elasticsearch)。 运行后,将正确的配置值传递给Jaeger收集器和查询服务。

#### 配置

**最小**

```bash
docker run \
  -e SPAN_STORAGE_TYPE=elasticsearch \
  -e ES_SERVER_URLS=<...> \
  jaegertracing/jaeger-collector:1.14
```

**所有选项**

要查看配置选项的完整列表,可以运行以下命令:

```bash
docker run \
  -e SPAN_STORAGE_TYPE=elasticsearch \
  jaegertracing/jaeger-collector:1.14 \
  --help
```

#### Elasticsearch索引的分片和副本

分片和副本是需要特别注意的一些配置值,因为这取决于 索引创建.进入[本文](https://qbox.io/blog/optimizing-elasticsearch-how-many-shards-per-index)查看 有关选择应选择多少个碎片进行优化的更多信息。

#### 升级到Elasticsearch 7.x

Elasticsearch 7中的索引映射与旧版本不向后兼容。 因此,将Elasticsearch 7与使用较旧版本创建的数据一起使用将不起作用。 Elasticsearch 6.8支持7.x,6.x,5.x兼容映射。升级必须完成 首先是ES 6.8,然后应用数据迁移或等到旧的每日索引删除\(这要求 以`--es.version = 7`开始Jaeger,以对新创建的索引强制使用ES 7.x映射。

Jaeger默认情况下使用Elasticsearch ping端点\(`/`\)导出所使用的版本 用于索引映射选择。该版本可以被标志`--es.version`覆盖。

#### 数据迁移

数据迁移必须在Elasticsearch 6.8上完成。

1.为Elasticsearch 7.x创建索引模板。使用标志`--es.version = 7`启动Jaeger收集器,或手动将模板提交给API。 2.使用新映射将所有范围和服务索引重新索引为索引。新索引的后缀为-1:

```text
curl -ivX POST -H "Content-Type: application/json" http://localhost:9200/_reindex -d @reindex.json
{
  "source": {
    "index": "jaeger-span-*"
  },
  "dest": {
    "index": "jaeger-span"
  },
  "script": {
    "lang": "painless",
    "source": "ctx._index = 'jaeger-span-' + (ctx._index.substring('jaeger-span-'.length(), ctx._index.length())) + '-1'"
  }
}
```

3.使用旧映射删除索引:

```text
curl -ivX DELETE -H "Content-Type: application/json" http://localhost:9200/jaeger-span-\*,-\*-1
```

4.创建不带有`-1`后缀的索引:

```text
curl -ivX POST -H "Content-Type: application/json" http://localhost:9200/_reindex -d @reindex.json
{
  "source": {
    "index": "jaeger-span-*"
  },
  "dest": {
    "index": "jaeger-span"
  },
  "script": {
    "lang": "painless",
    "source": "ctx._index = 'jaeger-span-' + (ctx._index.substring('jaeger-span-'.length(), ctx._index.length() - 2))"
  }
}
```

5.删除后缀索引:

```text
curl -ivX DELETE -H "Content-Type: application/json" http://localhost:9200/jaeger-span-\*-1
```

类似地为其他Jaeger索引运行命令。

可能存在更有效的迁移过程。请与社区分享任何发现。

### Kafka

自1.6.0起在Jaeger中受支持 支持的Kafka版本:0.9+

Kafka可用作收集器和实际存储之间的中间缓冲区。 收集器配置有`SPAN_STORAGE_TYPE = kafka`,使其可以将所有接收到的spans 写入Kafka topic。在版本1.7.0中添加的新组件[Ingester](https://www.jaegertracing.io/docs/1.14/deployment/#ingester)用于读取 Kafka并将span存储在另一个存储后端\(Elasticsearch或Cassandra\)中。

写入Kafka对于建立后处理数据管道特别有用。

#### 配置

**最小**

```bash
docker run \
  -e SPAN_STORAGE_TYPE=kafka \
  -e KAFKA_BROKERS=<...> \
  -e KAFKA_TOPIC=<...> \
  jaegertracing/jaeger-collector:1.14
```

**所有选项**

要查看配置选项的完整列表,可以运行以下命令:

```bash
docker run \
  -e SPAN_STORAGE_TYPE=kafka \
  jaegertracing/jaeger-collector:1.14 \
  --help
```

#### Topic & partitions

除非您的Kafka集群配置为自动创建topics,否则您需要提前创建topics。 您可以参考[Kafka快速入门文档](https://kafka.apache.org/documentation/#quickstart_createtopic)了解如何创建topics。

您可以在[官方文档](https://kafka.apache.org/documentation/#intro_topics)中找到有关主题和分区的更多信息. [本文](https://www.confluent.io/blog/how-to-choose-the-number-of-topicspartitions-in-a-kafka-cluster/)提供了有关如何选择分区数量的更多详细信息。

### 存储插件

Jaeger支持基于gRPC的存储插件。有关更多信息,请参考[jaeger/plugin/storage/grpc](https://github.com/jaegertracing/jaeger/tree/master/plugin/storage/grpc)

可用的插件:

\*[InfluxDB](https://github.com/influxdata/jaeger-influxdb/)

```bash
docker run \
  -e SPAN_STORAGE_TYPE=grpc-plugin \
  -e GRPC_STORAGE_PLUGIN_BINARY=<...> \
  -e GRPC_STORAGE_PLUGIN_CONFIGURATION_FINE=<...> \
  jaegertracing/all-in-one:1.14
```

## Ingester

**jaeger-ingester**是一项服务,可从Kafka主题读取跨度数据并将其写入另一个存储后端\(Elasticsearch或Cassandra\)。

| Port | Protocol | Function |
| :--- | :--- | :--- |
| 14270 | HTTP | Healthcheck at `/` and metrics at `/metrics` |

要查看所有公开的配置选项,请运行以下命令:

```bash
docker run \
  -e SPAN_STORAGE_TYPE=cassandra \
  jaegertracing/jaeger-ingester:{{< currentVersion >}}
  --help
```

## 查询服务和UI

jaeger-query服务于API端点和React/Javascript UI通信。 该服务是无状态的,通常在负载均衡器\(例如[ **NGINX** ](https://www.nginx.com/)\)之后运行。

在默认设置下,查询服务公开以下端口:

| Port | Protocol | Function |
| :--- | :--- | :--- |
| 16686 | HTTP | `/api/*` endpoints and Jaeger UI at `/` |
| 16687 | HTTP | Healthcheck at `/` and metrics at `/metrics` |

### 最小部署示例\(Elasticsearch后端\):

```bash
docker run -d --rm \
  -p 16686:16686 \
  -p 16687:16687 \
  -e SPAN_STORAGE_TYPE=elasticsearch \
  -e ES_SERVER_URLS=http://<ES_SERVER_IP>:<ES_SERVER_PORT> \
  jaegertracing/jaeger-query:{{< currentVersion >}}
```

### UI基本路径

所有**jaeger-query** HTTP路由的基本路径都可以设置为非根值,例如`/jaeger`将导致所有UI URL以`/jaeger`开头。 在反向代理后面运行`jaeger-query`时,这很有用。

基本路径可以通过`--query.base-path`命令行参数或`QUERY_BASE_PATH环`境变量来配置。

### UI自定义和嵌入

请参阅[专用前端/UI页面](frontend-ui.md)。

## 聚合任务和服务依赖

生产部署需要一个外部进程来聚合数据并在服务之间创建依赖关系链接 。项目[spark-dependencies](https://github.com/jaegertracing/spark-dependencies)是一个Spark作业, 可派生依赖项链接并将其直接存储到存储中。

## 配置

所有二进制文件都接受[viper](https://github.com/spf13/viper)和[cobra](https://github.com/spf13/cobra)库提供的命令行属性和环境变量。 请参阅[CLI 参数](cli.md)页面以获取更多信息。

