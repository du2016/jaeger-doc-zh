# 采样
Jaeger库实现了一致的前期(或基于头)的采样。例如,假设我们有一个简单的调用图,其中服务A调用服务B,
服务B调用服务C:`A-> B-> C`。当服务A收到不包含跟踪信息的请求时,Jaeger跟踪器将启动一个新的trace,
为其分配一个随机跟踪ID,并根据当前安装的采样策略做出采样决定。
采样决策将与请求一起传播到B和C,因此那些服务将不再再次做出采样决策,而是会尊重顶级服务A的决策。
这种方法保证了,如果对跟踪进行了采样,则所有其spans将记录在后端。
如果每个服务都做出自己的抽样决定,那么我们很少会在后端获得完整的跟踪。

## 客户端采样配置

当使用配置对象实例化跟踪器时,可以通过`sampler.type`和`sampler.param`属性选择采样类型.Jaeger库支持以下采样器:

* **常量**(`sampler.type=const`)采样器始终对所有traces做出相同的决定。
它要么采样所有跟踪(`sampler.param=1`),要么都不采样(`sampler.param=0`)。
* **概率** (`sampler.type=probabilistic`)采样器做出随机采样决策,
采样概率等于`sampler.param`属性的值。例如,在`sampler.param=0.1`的情况下,将在10条迹线中大约采样1条。
* **Rate Limiting**(`sampler.type=ratelimiting`)采样器使用漏斗速率限制器来确保以一定的恒定速率对轨迹进行采样。
例如,当`sampler.param=2.0`时,它将以每秒2条迹线的速率对请求进行采样。
* **远程**(`sampler.type=remote`,这也是默认值),采样器会向Jaeger代理咨询有关在当前服务中使用的适当采样策略。
这允许从Jaeger后端中的[中央配置](#收集器采样配置)甚至动态地控制服务中的采样策略
(请参阅[自适应采样](https://github.com/jaegertracing/jaeger/issues/365))。

### 自适应采样器

自适应采样器是一个组合了两个功能的复合采样器:

  * 它基于每个操作(即根据span操作名称做出抽样决策。这在API服务中特别有用,这些服务的端点的流量可能非常不同,
并且对整个服务使用单个概率采样器可能会使某些低QPS端点饿死(从不采样)。
  * 它支持最低保证的采样率,例如始终允许每秒最多N trace每秒钟,然后以一定概率对所有采样率进行采样(一切都是每次操作,不是每项服务)。

可以静态配置每个操作参数,也可以在远程采样器的帮助下从Jaeger后端定期提取每个操作参数。
自适应采样器旨在与Jaeger后端即将推出的[Adaptive Sampling](https://github.com/jaegertracing/jaeger/issues/365)功能配合使用。

## 收集器采样配置

收集器可以通过`--sampling.strategies-file`选项通过静态采样策略实例化(如果使用Remote sampler配置,
则将传播到相应的服务)。该选项需要一个已定义采样策略的json文件的路径。

示例`strategies.json`
```json
{
  "service_strategies":[
    {
      "service": "foo",
      "type": "probabilistic",
      "param": 0.8,
      "operation_strategies":[
        {
          "operation": "op1",
          "type": "probabilistic",
          "param": 0.2
        },
        {
          "operation": "op2",
          "type": "probabilistic",
          "param": 0.4
        }
      ]
    },
    {
      "service": "bar",
      "type": "ratelimiting",
      "param": 5
    }
  ],
  "default_strategy": {
    "type": "probabilistic",
    "param": 0.5
  }
}
```

`service_strategies`定义特定于服务的采样策略,`operation_strategies`定义特定于服务的采样策略。
可能有两种类型的策略:[probabilistic]和[ratelimiting],在上面已进行了描述(注意:`operation_strategies`不支持`ratelimiting`).
`default_strategy`定义了如果服务未包含在`service_strategies`中而传播的全部采样策略。

在上面的示例中,所有服务`foo`操作均以0.8的概率概率抽样,除了`op1`和`op2`的概率分别为0.2和0.4概率抽样。
服务`bar`的所有操作的速率限制为每秒5条trace。任何其他服务均以0.5的概率进行概率采样。
