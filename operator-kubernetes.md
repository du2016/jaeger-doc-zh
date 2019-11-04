# Kubernetes operator

# 了解operator

Jaeger operator是[Kubernetes operator](https://coreos.com/operators/)的实现。
operator是软件的一部分,可以减轻运行另一软件的操作复杂性。从技术上讲,**Operators**是一种打包,部署和管理Kubernetes应用程序的方法。

Kubernetes应用程序是既部署在Kubernetes上,
又使用Kubernetes API和`kubectl`(kubernetes)或`oc`(OKD)工具进行管理的应用程序。
为了充分利用Kubernetes,您需要扩展一组聚合 API以服务和管理在Kubernetes上运行的应用程序。
将Operators视为在Kubernetes上管理此类应用程序的运行时程序。

# 安装operator

> Jaeger Operator版本跟踪Jaeger组件(Query, Collector, Agent)的一种版本。
>当发布Jaeger组件的新版本时,将发布一个新版本的operator,该operator了解如何将先前版本的运行实例升级到新版本。

## 在Kubernetes上安装operator

以下说明将创建"`observability`命名空间并安装Jaeger Operator。

> 确保您的`kubectl`命令已正确配置为与有效的Kubernetes集群通信。
> 如果没有集群,则可以使用[`minikube`](https://kubernetes.io/docs/tasks/tools/install-minikube/)在本地创建一个群集。

要安装operator,请运行:

```
kubectl create namespace observability # <1>
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing_v1_jaeger_crd.yaml # <2>
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
kubectl create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
```
<1>这将在部署文件中创建默认使用的名称空间。
如果要将Jaeger运算符安装在其他名称空间中,
则必须编辑部署文件以将"可观察性"更改为所需的名称空间值。

<2>这将安装"Custom Resource Definitio" `apiVersion:jaegertracing.io/v1`。

此时,应该有一个`jaeger-operator`部署。您可以通过运行以下命令来查看它:

```
$ kubectl get deployment jaeger-operator -n observability

NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
jaeger-operator   1         1         1            1           48s
```

operator现在准备好用于创建Jaeger实例。

## 在OKD/OpenShift上安装operator

上一节中的说明也适用于在OKD或OpenShift上安装operator。
在安装基于角色的访问控制(RBAC)规则,自定义资源定义和operator时,请确保以特权用户身份登录。

```
oc login -u <privileged user>

oc new-project observability # <1>
oc create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing_v1_jaeger_crd.yaml # <2>
oc create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
oc create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
oc create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
oc create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
```
<1>这将在部署文件中创建默认使用的名称空间。
如果要将Jaeger运算符安装在其他名称空间中,则必须编辑部署文件以将`observability`更改为所需的名称空间值。

<2>这将为`apiVersion:jaegertracing.io/v1`安装"Custom Resource Definition"。

安装完operator后,将角色`jaeger-operator`授予应该能够安装单个Jaeger实例的用户。
以下示例创建一个role binding,允许用户`developer`创建Jaeger实例:

```
oc create \
  rolebinding developer-jaeger-operator \
  --role=jaeger-operator \
  --user=developer
```

授予角色后,切换回非特权用户。

＃快速入门 - 部署AllInOne镜像

创建Jaeger实例的最简单方法是通过创建YAML文件,如以下示例所示。这将安装默认的A
llInOne策略,默认情况下使用内存存储将`all-in-one`镜像(agent, collector, query, ingestor, Jaeger UI)部署在单个容器中。

> 此默认策略仅用于开发,测试和演示目的,而不用于生产。

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: simplest
```

然后可以将YAML文件与`kubectl`一起使用:

```
kubectl apply -f simplest.yaml
```

在几秒钟内,Jaeger的一个新的内存中all-in-one实例将可用,适用于快速演示和开发目的。要检查创建的实例,列出`jaeger`对象:

```
$ kubectl get jaegers
NAME        CREATED AT
simplest    28s
```

要获取容器名称,请查询属于`simplest` Jaeger实例的容器:


```
$ kubectl get pods -l app.kubernetes.io/instance=simplest
NAME                        READY     STATUS    RESTARTS   AGE
simplest-6499bb6cdd-kqx75   1/1       Running   0          2m
```

同样,可以使用从上一个示例获得的容器名称直接从容器中查询日志,也可以从属于我们实例的所有容器中查询日志:

```
$ kubectl logs -l app.kubernetes.io/instance=simplest
...
{"level":"info","ts":1535385688.0951214,"caller":"healthcheck/handler.go:133","msg":"Health Check state change","status":"ready"}
```

> 在OKD/OpenShift上,必须指定容器名称。

```
$ kubectl logs -l app.kubernetes.io/instance=simplest -c jaeger
...
{"level":"info","ts":1535385688.0951214,"caller":"healthcheck/handler.go:133","msg":"Health Check state change","status":"ready"}
```

# 部署策略

创建Jaeger实例时,它与策略相关联。
该策略在自定义资源文件中定义,并确定要用于Jaeger后端的体系结构。默认策略是`allInOne`。其他可能的值是`production`和`streaming`。

以下各节介绍了可用的策略。

## AllInOne(默认)策略

该策略旨在用于开发,测试和演示目的。

主要的后端components, agent, collector, query都打包到单个可执行文件中,该可执行文件被配置为(默认情况下)使用内存存储。

## 生产策略

`生产`策略旨在(顾名思义)用于生产环境,在该环境中,长期存储跟踪数据非常重要,并且需要更具可伸缩性和高可用性的体系结构。
因此,每个后端组件都是单独部署的。

agent可以作为辅助工具注入到已检测应用程序中,也可以作为daemonset注入。

查询和收集器服务配置有受支持的存储类型-当前为Cassandra或Elasticsearch。可以根据需要提供这些组件中每个组件的多个实例,以实现性能和弹性。

主要的附加要求是提供存储类型和存储选项的详细信息,例如:

```
    storage:
      type: elasticsearch
      options:
        es:
          server-urls: http://elasticsearch:9200

```

## 流策略

通过提供有效地位于收集器和后端存储(Cassandra或Elasticsearch)之间的流功能,`流`策略旨在增强`生产`策略。
这具有减轻高负载情况下后端存储压力的好处,并使其他跟踪后处理功能可以直接从流传输平台(Kafka)利用实时span数据。

唯一需要的附加信息是提供访问Kafka平台的详细信息,该平台在`collector`组件(作为生产者)和`ingester`组件(作为消费者)中配置:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: simple-streaming
spec:
  strategy: streaming
  collector:
    options:
      kafka: # <1>
        producer:
          topic: jaeger-spans
          brokers: my-cluster-kafka-brokers.kafka:9092
  ingester:
    options:
      kafka: # <1>
        consumer:
          topic: jaeger-spans
          brokers: my-cluster-kafka-brokers.kafka:9092
      ingester:
        deadlockInterval: 0 # <2>
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
```
<1>确定kafka的配置用于collector生产消息,ingester消费消息。

<2>可以禁用死锁间隔,以防止在默认的1分钟内没有消息到达时终止ingester

> 可以使用[Strimzi的Kafka运算符](https://strimzi.io/)来配置Kafka环境。

# 了解Custom Resource Definitions

在Kubernetes API中,资源是一个端点,
用于存储某种类型的API对象的集合。例如,内置的Pods资源包含Pod对象的集合。
 _Custom Resource Definition_(CRD)对象在集群中定义了一个新的唯一对象`Kind`,并使Kubernetes API服务器处理其整个生命周期。

要创建_Custom Resource_(CR)对象,群集管理员必须首先创建一个Custom Resource Definition(CRD),
CRD允许群集用户创建CR,以将新资源类型添加到他们的项目中。操作员监视要创建的自定义资源对象,
并且在看到正在创建的自定义资源时,它会根据在自定义资源对象中定义的参数来创建应用程序。

> 虽然只有集群管理员才能创建CRD,但开发人员可以在现有CRD拥有读写权限的情况下从其创建CR。


供参考,以下是创建更复杂的all-in-one实例的方法:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: my-jaeger
spec:
  strategy: allInOne # <1>
  allInOne:
    image: jaegertracing/all-in-one:latest # <2>
    options: # <3>
      log-level: debug # <4>
  storage:
    type: memory # <5>
    options: # <6>
      memory: # <7>
        max-traces: 100000
  ingress:
    enabled: false # <8>
  agent:
    strategy: DaemonSet # <9>
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: "" # <10>
```

<1>默认策略是`allInOne`。其他可能的值是`production`和`streaming`。

<2>以常规Docker语法使用的镜像。

<3>(与存储无关的)选项逐字传递给基础二进制文件。有关所有可用选项,请参阅Jaeger文档和/或相关二进制文件中的`--help`选项。

<4>该选项是一个简单的`key: value`映射。在这种情况下,我们希望将选项`--log-level=debug`传递给二进制文件。

<5>要使用的存储类型。默认情况下,它将是`memory`,但是可以是任何其他受支持的存储类型(Cassandra,Elasticsearch,Kafka)。

<6>所有与存储相关的选项应放在此处,而不是在`allInOne`或其他组件选项下。

<7>一些选项带有名称空间,我们可以将它们分成嵌套对象。我们可以指定`memory.max-traces:100000`。

<8>默认情况下,将为查询服务创建一个入口对象。可以通过将其启用选项设置为false来禁用它。如果在OpenShift上部署,则将由Route对象表示。

<9>默认情况下,操作员假定代理已在目标pod内作为边车部署。将策略指定为`DaemonSet`会更改,并使operator将代理部署为DaemonSet。
请注意,您的跟踪器客户端可能必须重写`JAEGER_AGENT_HOST`环境变量才能使用节点的IP。

<10>定义要应用于所有部署(而非服务)的注释。这些可以被各个组件上定义的注释所覆盖。

您可以在[GitHub上]((https://github.com/jaegertracing/jaeger-operator/tree/master/deploy/examples))查看针对不同Jaeger配置的示例
自定义资源。

# 配置 Custom Resource

您可以使用最简单的示例(如上所示)并使用默认值创建Jaeger实例,也可以创建自己的custom resource文件。

## 存储选项

### Cassandra存储

当存储类型设置为Cassandra时,operator将自动创建一个批处理作业,
该作业创建Jaeger运行所需的架构。该批处理作业将阻止Jaeger安装,因此仅在成功创建模式后才开始。
可以通过将enabled属性设置为false来禁用该batch job的创建:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: cassandra-without-create-schema
spec:
  strategy: allInOne
  storage:
    type: cassandra
    cassandraCreateSchema:
      enabled: false # <1>
```
<1>默认为`true`

batch job的其他方面也可以配置。带有所有可能选项的示例如下所示:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: cassandra-with-create-schema
spec:
  strategy: allInOne # <1>
  storage:
    type: cassandra
    options: # <2>
      cassandra:
        servers: cassandra
        keyspace: jaeger_v1_datacenter3
    cassandraCreateSchema: # <3>
      datacenter: "datacenter3"
      mode: "test"
```
<1>`生产`和`流`的工作原理相同。

<2>这些选项用于常规的Jaeger组件,例如`collector`和`query`。

<3>`create-schema`作业的选项。

> 默认的create-schema作业使用`MODE = prod`,这意味着使用NetworkTopologyStrategy作为类的副本数为2,
> 这实际上意味着在Cassandra集群中至少需要3个节点。如果需要`SimpleStrategy`,则将模式设置为`test`,然后将副本数设置为`1`。
>有关更多详细信息,请参考[create-schema脚本](https://github.com/jaegertracing/jaeger/blob/master/plugin/storage/cassandra/schema/create.sh)。

### Elasticsearch存储

在某些情况下,Jaeger操作员可以使用[Elasticsearch Operator](https://github.com/openshift/elasticsearch-operator)来提供合适的Elasticsearch集群。

> 仅在OpenShift群集上测试了此功能。
>此功能不支持Spark依赖项[问题＃294](https://github.com/jaegertracing/jaeger-operator/issues/294)。

当Jaeger生产实例中没有`es.server-urls`选项并且将elasticsearch设置为存储类型时,
Jaeger Operator将通过创建基于以下内容的自定义资源,通过Elasticsearch Operator创建Elasticsearch集群。
存储部分中提供的配置.Elasticsearch集群专用于单个Jaeger实例。

可以通过将标志`--es-provision`设置为false来禁用Elasticsearch集群的自我配置。默认值为`auto`,
这将使Jaeger Operator可以查询Kubernetes集群以处理'Elasticsearch'自定义资源。
这通常是由Elasticsearch Operator在其安装过程中设置的,
因此,如果预期Elasticsearch Operator在Jaeger Operator之后运行,则可以将该标志设置为true。

> 目前,每个命名空间只能有一个Jaeger,具有自配置的Elasticsearch实例。

#### Elasticsearch索引清理器作业

默认情况下,当使用`elasticsearch`存储时,将创建一个作业以清除其中的旧traces,下面列出了其选项,因此您可以将其配置为用例

```
storage:
  type: elasticsearch
  esIndexCleaner:
    enabled: false                                //turn the job deployment on and off
    numberOfDays: 7                               //number of days to wait before deleting a record
    schedule: "55 23 * * *"                       //cron expression for it to run
    image: jaegertracing/jaeger-es-index-cleaner  //image of the job
```

## 派生依赖

派生依赖项的处理将从存储中收集跨度,分析服务之间的链接,并将其存储以供以后在UI中呈现。
该作业只能与`生产`策略和存储类型`cassandra`或`elasticsearch`一起使用。

```
storage:
  type: elasticsearch
  dependencies:
    enabled: true                                 //turn the job deployment on and off
    schedule: "55 23 * * *"                       //cron expression for it to run
    sparkMaster:                                  //spark master connection string, when empty spark runs in embedded local mode
```

到存储的连接配置是从存储选项派生的。

##自动注入Jaeger Agent Sidecars

如果部署的注解`sidecar.jaegertracing.io/inject`具有适当的值,
那么operator可以将Jaeger Agent sidecar注入`Deployment`工作负载中。值可以是`true`(作为字符串),
也可以是由`Kubectl get jaegers`返回的Jaeger实例名称。当使用`true`时,
与部署相同的名称空间应完全有一个Jaeger实例,否则,operator将无法自动确定要使用哪个Jaeger实例。

以下代码片段显示了一个简单的应用程序,该应用程序将注入sidecar,Jaeger Agent指向同一命名空间中可用的单个Jaeger实例:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    "sidecar.jaegertracing.io/inject": "true" # <1>
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: acme/myapp:myversion
```

<1>`true`(作为字符串)或Jaeger实例名称。

完整的示例部署可在[deploy/examples/business-application-injected-sidecar.yaml](https://github.com/jaegertracing/jaeger-operator/blob/master/deploy/examples/business-application-injected-sidecar.yaml)中找到。

## 将agent安装为DaemonSet

默认情况下,operator希望将agent作为sidecars部署到目标应用程序。
这对于多种目的很方便,例如在多租户场景中或具有更好的负载平衡,
但是在某些场景中,您可能希望将代理安装为`DaemonSet`。在这种情况下,将`agent`的策略指定为`DaemonSet`,如下所示:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: my-jaeger
spec:
  agent:
    strategy: DaemonSet
```

> 如果您尝试使用DaemonSet作为策略在同一集群上安装两个Jaeger实例,
> 由于代理需要绑定到节点上的指定端口,因此只有一个最终将部署DaemonSet。因此,第二个守护程序集将无法绑定到那些端口。

然后很可能需要告知您的tracer客户端 agent位于何处。通常通过将环境变量`JAEGER_AGENT_HOST`设置为Kubernetes节点的IP值来完成,例如:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: acme/myapp:myversion
        env:
        - name: JAEGER_AGENT_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP

```

### OpenShift

在OpenShift中,只有在设置了特殊的安全上下文时才能设置`HostPort`.
Jaeger代理可以使用一个单独的服务帐户来绑定到`HostPort`,如下所示:

```
oc create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/examples/openshift/hostport-scc-daemonset.yaml＃<1>
oc new-project myappnamespace
oc create -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/examples/openshift/service_account_jaeger-agent-daemonset.yaml＃<2>
oc adm policy add-scc-to-user daemonset-with-hostport -z jaeger-agent-daemonset＃<3>
oc apply -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/examples/openshift/agent-as-daemonset.yaml＃<4>
```
<1>具有`allowHostPorts`策略的`SecurityContextConstraints`

<2> Jaeger代理程序要使用的`ServiceAccount`

<3>将安全策略添加到服务帐户

<4>使用在以上步骤中创建的`serviceAccount`创建Jaeger实例

> 如果没有这样的策略,则类似以下错误将阻止创建`DaemonSet`:\
> `Warning FailedCreate 4s (x14 over 45s) daemonset-controller Error creating: pods "agent-as-daemonset-agent-daemonset-" is forbidden: unable to validate against any security context constraint:[spec.containers[0].securityContext.containers[0].hostPort: Invalid value: 5775: Host ports are not allowed to be used`

几秒钟后,`DaemonSet`应该启动并运行:

```
$ oc get daemonset agent-as-daemonset-agent-daemonset
NAME                                 DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE
agent-as-daemonset-agent-daemonset   1         1         1           1            1
```

## Secrets支持

operator支持将机密传递给Collector, Query and All-In-One deployment。例如,
这可以用于传递凭据(用户名/密码)以访问基础存储后端(例如:Elasticsearch)。
这些secrets可以在(Collector/Query/All-In-One)节点中作为环境变量使用。

```
    storage:
      type: elasticsearch
      options:
        es:
          server-urls: http://elasticsearch:9200
      secretName: jaeger-secrets
```

secret本身将在`jaeger-operator`定制资源之外进行管理。

## 配置UI

可以在[here](./frontend-ui.md#configuration)中找到以json格式定义的有关UI各种配置选项的信息。

要在Custom Resource中应用UI配置更改,可以以yaml格式包含相同的信息,如下所示:

```
    ui:
      options:
        dependencies:
          menuEnabled: false
        tracking:
          gaID: UA-000000-2
        menu:
        - label: "About Jaeger"
          items:
            - label: "Documentation"
              url: "https://www.jaegertracing.io/docs/latest"
        linkPatterns:
        - type: "logs"
          key: "customer_id"
          url: /search?limit=20&lookback=1h&service=frontend&tags=%7B%22customer_id%22%3A%22#{customer_id}%22%7D
          text: "Search for other traces for customer_id=#{customer_id}"
```

##定义采样策略

> 如果跟踪是由Istio代理启动的,那么这是不相关的,
> 因为Istio代理在此处做出了采样决策。而且只有在使用Jaeger tracer(client).时,Jaeger采样决策才有意义。

operator可用于定义将提供给已配置为使用远程采样器的跟踪器的采样策略:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: with-sampling
spec:
  strategy: allInOne
  sampling:
    options:
      default_strategy:
        type: probabilistic
        param: 50
```

本示例定义了默认采样策略的概率,有50％的机会采样trace实例。

请参阅[收集器采样配置](https://www.jaegertracing.io/docs/latest/sampling/#collector-sampling-configuration)上的Jaeger文档,
以了解如何配置服务和端点采样。该文档中描述的JSON表示形式可以通过转换为YAML在运算符中使用。

##更细粒度的配置

custom resource可用于定义应用于所有Jaeger组件或单个组件级别的更细粒度的Kubernetes配置。

当需要通用定义(用于所有Jaeger组件)时,可以在`spec`节点下定义它。
当定义与单个组件相关时,将其放置在`spec/<component>`节点下。

支持的配置类型包括:

*[亲和性](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)确定可以将Pod分配给哪些节点

*[注释](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

*[标签](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

*[资源](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container)以限制cpu和内存

*[污点](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)与`taints'结合使用,以使Pod避免被节点排斥

*[卷](https://kubernetes.io/docs/concepts/storage/volumes/)和卷安装

*[serviceAccount](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)以独立的身份运行每个组件

*[securityContext](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)定义运行组件的特权

```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: simple-prod
spec:
  strategy: production
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
  annotations:
    key1: value1
  labels:
    key2: value2
  resources:
    requests:
      memory: "64Mi"
      cpu: "250m"
    limits:
      memory: "128Mi"
      cpu: "500m"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  tolerations:
    - key: "key1"
      operator: "Equal"
      value: "value1"
      effect: "NoSchedule"
    - key: "key1"
      operator: "Equal"
      value: "value1"
      effect: "NoExecute"
  serviceAccount: nameOfServiceAccount
  securityContext:
    runAsUser: 1000
  volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
```

# 访问Jaeger控制台(UI)

## Kubernetes

operator创建了Kubernetes[ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)路由,
这是Kubernetes将服务暴露于外界的标准,但默认情况下它不随Ingress提供程序一起提供。
查看[Kubernetes文档](https://kubernetes.github.io/ingress-nginx/deploy/#verify-installation),
以获得最适合您平台的Ingress提供程序的方法。以下命令在`minikube`上启用Ingress提供程序:

```
minikube addons enable ingress
```

启用Ingress后,可以通过查询Ingress对象找到Jaeger控制台的地址:

```
$ kubectl get ingress
NAME             HOSTS     ADDRESS          PORTS     AGE
simplest-query   *         192.168.122.34   80        3m
```

在此示例中,Jaeger UI可从[http://192.168.122.34]( http://192.168.122.34)访问。

## OpenShift

当操作员在OpenShift上运行时,操作员将自动为查询服务创建一个`Route`对象。使用以下命令检查主机名/端口:

```
oc get routes
```

> 确保将`https`与您从上述命令中获得的主机名/端口一起使用,否则您将看到类似`未提供应用程序`的消息。

默认情况下,Jaeger UI受OpenShift的OAuth服务保护,
任何有效用户都可以登录。要禁用此功能并使Jaeger UI处于不安全状态,请在自定义资源文件中将Ingress属性`security`设置为`none`:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: disable-oauth-proxy
spec:
  ingress:
    security: none
```

可以在`.Spec.Ingress.OpenShift.SAR`和`.Spec.Ingress.Openshift.DelegateURLs`中指定自定义的`SAR`和`DELETEURL`值,如下所示:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: custom-sar-oauth-proxy
spec:
  ingress:
    openshift:
      sar: '{"namespace": "default", "resource": "pods", "verb": "get"}'
      delegateUrls: '{"/":{"namespace": "default", "resource": "pods", "verb": "get"}}'
```    

设置好`delegateUrls`后,Jaeger operator需要在UI代理使用的服务帐户(`{InstanceName}-ui-proxy`)
和角色`system:auth-delegator`之间创建一个新的`ClusterRoleBinding`,
根据OpenShift OAuth代理的要求。因此,operator本身使用的服务帐户需要具有相同的群集角色绑定。
为此,必须创建如下的`ClusterRoleBinding`:

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jaeger-operator-with-auth-delegator
  namespace: observability
subjects:
- kind: ServiceAccount
  name: jaeger-operator
  namespace: observability
roleRef:
  kind: ClusterRole
  name: system:auth-delegator
  apiGroup: rbac.authorization.k8s.io
```

群集管理员不愿意让用户使用该群集角色来部署Jaeger实例,
可以自由的不将此群集角色添加到operator的服务帐户中。
在这种情况下,操作员将自动检测到缺少所需的权限,
并将记录类似于以下消息:`请求的实例为OAuth代理指定了proxyUrls选项,但该操作员无法为其分配适当的群集角色(系统:auth-delegator)。
在操作员的服务帐户和群集角色'system:auth-delegator'之间创建群集角色绑定,以允许实例使用"delegateUrls"`。

Jaeger操作员还支持通过OpenShift OAuth代理使用htpasswd文件进行身份验证。
要使用它,请在特定于OpenShift的条目中指定htpasswdFile选项,
指向本地磁盘中的文件htpasswd文件位置。可以使用htpasswd实用程序创建htpasswd文件:

```
$ htpasswd -cs /tmp/htpasswd jdoe
New password:
Re-type new password:
Adding password for user jdoe
```

然后,该文件可用作`kubectl create secret`命令的输入:

```
$ kubectl create secret generic htpasswd --from-file=htpasswd=/tmp/htpasswd
secret/htpasswd created
```

创建密钥后,可以在Jaeger CR中将其指定为volume/volume mount:

```
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: with-htpasswd
spec:
  ingress:
    openshift:
      sar: '{"namespace": "default", "resource": "pods", "verb": "get"}'
      htpasswdFile: /usr/local/data/htpasswd
  volumeMounts:
  - name: htpasswd-volume
    mountPath: /usr/local/data
  volumes:
  - name: htpasswd-volume
    secret:
      secretName: htpasswd
```

# 升级operator及其托管实例

每个Jaeger Operator版本都遵循一个Jaeger版本。每当安装新版本的Jaeger Operator时,
该操作员管理的所有Jaeger实例都将升级到该操作员支持的版本

可以通过更改部署(`kubectl编辑部署jaeger-operator`)或通过诸如[Operator Lifecycle Manager(OLM)](https://github.com/operator-framework/操作员生命周期经理)。

# 更新Jaeger实例(实验性)

Jaeger实例可以通过更改`CustomResource`来更新,方法是通过`kubectl edit jaeger simplest`,
其中`simplest`是Jaeger的实例名称,或者通过`kubectl apply -f simplest.yaml`应用更新的YAML文件。

> Jaeger实例的名称无法更新,因为它是资源标识信息的一部分。

可以应用更简单的更改(例如更改副本大小)而不必担心,而应密切注意策略的更改,这可能会导致单个组件(collector/query/agent)中断。

虽然支持更改后备存储,但不支持数据迁移。

# 删除Jaeger实例

要删除实例,请在创建实例时使用`delete`命令和自定义资源文件:

```
kubectl delete -f simplest.yaml
```

另外,您可以通过运行以下命令来删除Jaeger实例:

```
kubectl delete jaeger simplest
```

> 删除实例不会从该实例使用的任何永久性存储中删除数据。但是,来自内存中实例的数据将丢失。

# 监控operator

Jaeger Operator在0.0.0.0:8383/metrics上使用内部指标启动Prometheus兼容端点,该指标可用于监视进程。


> Jaeger操作员尚未发布自己的指标。而是,它使由其使用的组件(例如Operator SDK)报告的可用指标。


# 卸载operator

使用以下命令卸载jaeger operator

```
kubectl delete -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/operator.yaml
kubectl delete -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role_binding.yaml
kubectl delete -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/role.yaml
kubectl delete -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/service_account.yaml
kubectl delete -f https://raw.githubusercontent.com/jaegertracing/jaeger-operator/master/deploy/crds/jaegertracing_v1_jaeger_crd.yaml
```
