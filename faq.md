---
# 常见问题

回答有关Jaeger的一些常见问题。

##为什么依赖项页面为空？

依赖关系页面显示了Jaeger跟踪的服务及其之间的连接的图表。当您在内存中使用"多合一"二进制文件时,将根据内存中存储的所有迹线按需计算图形。但是,如果您使用的是真正的分布式存储(如Cassandra或Elasticsearch),则扫描数据库中的所有数据以构建服务图的成本太高。相反,Jaeger项目提供了"大数据"作业,可用于从跟踪中提取服务图数据:

*[https://github.com/jaegertracing/spark-dependencies](https://github.com/jaegertracing/spark-dependencies)-可以定期运行的旧版Spark作业
*[https://github.com/jaegertracing/jaeger-analytics](https://github.com/jaegertracing/jaeger-analytics)-新的(实验性)流式Flink作业连续运行并以较小的时间间隔构建服务图

##为什么在Jaeger中看不到任何跨度？

请参阅[故障排除](./troubleshooting.md)指南。
