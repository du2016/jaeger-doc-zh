# 命令行参数
这是Jaeger二进制文件支持的CLI标志的自动生成的文档。

* 一些二进制文件的CLI标志根据`SPAN_STORAGE_TYPE`环境变量而改变.相关变化如下。
* 有些二进制文件支持_commands_(主要是信息性的),例如`env`,`docs`和`version`.这些命令不在此处。
* 还可以通过环境变量提供所有参数,方法是将所有字母更改为大写字母,并用下划线_替换所有标点符号.例如,可以通过环境变量" CASSANDRA_CONNECTIONS_PER_HOST"提供标志" --cassandra.connections-per-host"的值。

[官方命令行文档](https://www.jaegertracing.io/docs/1.14/cli/)
