> MySQL 调优主要是查询优化、数据量太大的分库分表优化和一些特殊场景优化。

## 查询优化

#### 1. 查询语句优化

这部分优化效果效果很微弱，但是却是开发人员写代码的基本功。

1. 尽量避免 select *：
   * 增加查询分析器解析成本
   * 增减字段容易与 resultMap 配置不一致
   * 无用字段增加网络消耗，尤其是 text 类型的字段
   * 索引覆盖，减少回表

1. 尽量避免使用 or 来连接，如果一个有索引，一个没有索引会导致放弃索引全表扫描，可以改成：union all
1. 慎用 in 和 not in，容易导致全表扫描，可以用 between、exists 代替
1. 尽量避免大事务操作，提高系统并发能力。
1. 尽量避免大数据量查询，可以考虑分批查询
1. update 语句如果只改 1、2 个字段不要 update 所有字段，会带来性能消耗和大量日志
1. 连表尽量用小表驱动大表

#### 2. 索引优化

1. 查询结果避免全表扫描，合理选择区分度高的列建立索引

1. 索引不是越多越好，修改操作需要维护甚至重建索引，降低效率。尽量扩展索引，而不是新建索引

1. 如果可以尽量使用联合索引，因为**联合索引比走单个索引效率高**，原因是减少扫描次数减少回表操作：比如走两个单字段索引，先根据 a 索引找到 10 个值，再根据 b 索引找到这 10 个值的 5 个，此时需要回表 15 次，联合索引根据两个条件确定扫描区间后扫描 5 个即可完成查找。

1. 合理使用联合索引，把区分度高的放在左边，减少扫描区间；

1. 尽量避免索引失效：
   1. 联合索引和 like 查询遵从最左匹配原则：先根据联合索引的第一个字段排序，排序后相同的数据再根据下一个字段排序，类推...所以联合索引的最左字段是可以走索引的，后面的不行。有左模糊需求可以考虑全文检索或 es，范围查询字段最好放到最后。
   1. 不要对 where 字段列进行表达式操作，比如：where num / 2 = 100，可以改为：where num = 100 * 2
   1. 尽量避免对 where 字段进行函数操作，可以尝试其他写法或在代码中处理。
   1. 字段类型不同造成的隐式转换
   1. 优化器认为全表扫描比走索引快时不走索引，比如索引区分度不高

#### 3. 建表优化
1. 尽量选择合适的数据类型，原因：占用空间小一次索引页查询能查到更多的数据，减少 IO，而且磁盘和内存的消耗更小。
1. 最好不要设置字段为 NULL，在一些特殊场景可能会导致索引失效，导致全表扫描。比如 where name is null
1. 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。

> <B>为什么数据库中不建议使用 NULL ？</B>
> <br><br>
> **MySQL 官网文档**：MySQL 难以优化引用可空列查询，它会使索引、索引统计和值更加复杂。可空列需要更多的存储空间，还需要 MySQL 内部进行特殊处理。可空列被索引后，每条记录都需要一个额外的字节，还能导致 MYisam 中固定大小的索引变成可变大小的索引。

## 大数据优化
当数据量达到一定大小的时候，普通的查询优化已经没有效果了，只能采取分而治之的思想将流量拆分（读写分离）或数据拆分（分库分表）。

#### 1. 为什么要分库分表？

当数据量小的时候，单库单表就能完成。

当数据增大时，通过数据库集群部署，**读写分离**，分摊主库读压力，但解决不了写压力。此时可以**解决数据库读压力**，理论上可以无限增加机器分摊读压力，只需要解决**主从延迟问题**。

当数据进一步增大时，单表数据太多，读取性能越来越差，读写分离解决不了这个问题，可以通过**分表**来拆分单表数据，根据一定拆分规则，理论上可以无限拆分。

但是分表只能解决单表查询性能问题，不能解决 QPS 过高，超过数据库负载的问题，这时候可以通过**分库**，把数据存在不同的库里，来分摊高于数据库的请求，也解决了读写分离的写压力问题。

读写分离、分库分表也增加了**可用性**。读写分离冗余了副本，一个节点挂了，其他节点能继续提供服务和备份。分库分表把数据放在多个机器，一个机器挂了只会有部分不可用或数据丢失，不会导致所有服务不可用和恢复所有数据。

#### 2. 什么时候分库？什么时候分表？

1. 当单表的数据过大时，选择分表
1. 当数据库读写 QPS 过高，数据库连接数不足时，选择分库
1. 当单表的数据库过大 + 数据库读写 QPS 过高，数据库连接数不足时，选择分库分表

#### 3. 如果需要分表，分多少张表合适？

需要结合业务来考虑，技术是为业务服务的。

了解已有数据库，根据业务每日增长或其他维度预估未来几年（系统往往 5 年会经历重构）数据会达到的数量，再加一些预留空间，每张表可以按 500w 计算表数量。

#### 4. 如果需要分库，分多少库合适？

结合业务和历史 QPS、RT，可以用历史 QPS 除以单机最高 QPS，比如历史最高 QPS 是 3500，单库能承受 1000 QPS，就可以分 4 个库。

#### 5. 分库分表依据？

1. 垂直拆分：

   * **垂直分库**：根据业务把大系统分成多个子系统，一般根据微服务来拆分，一个微服务一个数据库。

   * **垂直分表**：把一个表根据列字段分成多个表，比如增加扩展表，把影响性能的大字段单独拆出去。

1. 水平拆分：

   * **水平分库**：根据业务选择拆分规则，把一张表中的数据拆分到多张表中。

   * **水平分表**：根据业务选择拆分规则，把数据放到不同库里。

2. 常见拆分规则：

   * 根据范围（Range）拆分
   * 根据 hash 取模，适合符合单调性的数据
   * 地理区域拆分，适合根据区域产生的数据
   * 按照时间拆分，适合因为时间导致冷热的数据

#### 6. 分库分表后会有哪些问题？

1. **失去了本地事务支持**：可以考虑通过分布式事务解决方案解决。

1. **多库结果集合并**：查询的数据可能在多个数据库，需要查出多个库里的数据合并后返回，可能涉及到排序。

1. **跨库 JOIN**：尽量分成多个 SQL 语句查询，再在代码层处理。

#### 7. 有哪些中间件实现方案？

1. 其中基于代理方式的有 MySQL Proxy 和 Amoeba，
1. 基于 Hibernate 框架的是 Hibernate Shards，
1. 基于 jdbc 的有当当 sharding-jdbc，
1. 基于 mybatis 的类似 maven 插件式的有蘑菇街的蘑菇街 TSharding，
1. 通过重写 spring 的 ibatis template 类的Cobar Client。

## 常见优化案例

#### 1. 通用优化方案：
1. 接入搜索
1. 使用缓存
1. 空间换时间，比如聚合需要的数据再存一张新表

#### 2. 大数据量深度分页优化

深度分页偏移量越大，查询速度越慢，因为要从第一条满足条件的数据往后扫描。

**解决办法**：
1. 使用子查询优化
1. 去掉分页页码，用户很难点到很深的页

```
## 使用子查询优化
# 比如查询偏移量为 100 万的 10 条数据
SELECT * FROM `table` LIMIT 1000000, 10;

# 可以改成先查出第 100 万条那条数据，再往后扫描 10 条数据；这种只适合查询的字段递增的情况
SELECT * FROM `table` WHERE id >= (SELECT id FROM `table` LIMIT 1000000, 1) LIMIT 10;
# 不是递增的可以改成 in 查询
SELECT * FROM `table` WHERE id IN (SELECT t.id FROM (SELECT id FROM `table` LIMIT 1000000, 10) AS t)
```

#### 3. 大事务导致的性能问题优化

大事务特点是执行时间长，锁定数据大，容易拖慢其他正常查询。应该尽量避免大事务，降低事务加锁时间。

1. 把大事务分成多个小事务，比如分批处理等；
1. 把大事务里跟修改无关的操作放到事务外，减少事务执行时间。

## 不加条件或条件范围大分页查询，扫描行数过大优化

业务允许的话限制搜索时间范围，查最近三个月的数据，或其他业务允许的可以缩小范围的条件

## 执行计划（explain）
* **id**：各个子查询的执行顺序

* **select_type**：查询类型
   1. SIMPLE：不包含任何子查询或union等查询
   1. PRIMARY：包含子查询最外层查询就显示为 PRIMARY
   1. SUBQUERY：在 select 或 where 字句中包含的查询
   1. DERIVED：from 字句中包含的查询
   1. UNION：出现在union后的查询语句中
   1. UNION RESULT：从UNION中获取结果集，例如上文的第三个例子

* **table**：查询的数据表
* **partitions**：表分区
* **type**：访问类型
   1. ALL   扫描全表数据
   1. index 遍历索引
   1. range 索引范围查找
   1. index_subquery 在子查询中使用 ref
   1. unique_subquery 在子查询中使用 eq_ref
   1. ref_or_null 对Null进行索引的优化的 ref
   1. fulltext 使用全文索引
   1. ref   使用非唯一索引查找数据
   1. eq_ref 在join查询中使用PRIMARY KEYorUNIQUE NOT NULL索引关联。
   1. const 使用主键或者唯一索引，且匹配的结果只有一条记录。
   1. system const 连接类型的特例，查询的表为系统表。

* **possible_keys**：可能使用的索引
* **key**：实际使用的索引
* **key_length**：索引长度
* **ref**：上述表的连接匹配条件
* **rows**：估算的结果集数目
* **extra**：扩展信息

</details>