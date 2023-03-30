# 自定义flink es source

flink 官方不支持es source，只提供了es sink，可能是因为flink主要处理在线数据，而es是离线数据，很少有这样的场景，但有时候确实会有这种场景。比如安全行业，很多日志都是写入到es里，但es又无法做复杂的分析，而日志对实时性要求又不是很高，而且写入es的数据往往是经过处理的，flink直接解析原始数据工作量很大，这个时候就需要es source了。

数据不断写入es，es source每次处理前5分钟的离线数据，相当于数据整体往后推了5分钟的准实时效果。

#### 实现思路
1. 每5分钟采集一次，每次从es里根据时间字段查出过去5分钟的离线数据
2. 

#### 需要解决的问题
1. 需要知道数据处理到哪了，防止重复数据
2. 需要处理异常情况，记录异常位置重新消费

#### 实现流程
1. 继承`RichSourceFunction`，在`run`里读取es数据

#### 参考
* 自定义source官网说明：https://nightlies.apache.org/flink/flink-docs-release-1.13/docs/dev/table/sourcessinks/

* 自定义flink es source：https://blog.csdn.net/lw277232240/article/details/88739931