## Redis 官方文档
<br>
<details>
<summary>
点击展开 <a href="https://redis.io/docs/" target="_blank">Redis 官方文档</a> - 思维导图
</summary>
<img src="http://cdn.liufq.com/Fo4DKEpeWbbY2HoE9c6IbPvAK8Cu"/>
</details>

### Redis 简介
Redis 是一个开源的内存数据结构存储，**用作**数据库、缓存、消息代理和流引擎。Redis 提供了多种**数据结构**，例如字符串、散列、列表、集合、带范围查询的排序集合、位图、超日志、地理空间索引和流。Redis **内置**了复制、Lua 脚本、LRU 驱逐、事务和不同级别的磁盘持久性，并通过以下方式提供**高可用**性：Redis Sentinel 和 Redis Cluster 的自动分区。

### Redis 数据类型
#### 1. String
1. Redis String 是二进制安全的，这意味着 Redis String 可以包含任何类型的数据，例如 JPEG 图像或序列化的 Java 对象。

1. String 的最大长度为 512 MB。

1. 应用场景：
   * 使用 INCR 系列中的命令将字符串用作原子计数器：INCR、DECR、INCRBY。
   * 使用 APPEND 命令附加到字符串。
   * 使用字符串作为带有 GETRANGE 和 SETRANGE 的随机访问向量。
   * 在很小的空间内编码大量数据，或者使用GETBIT和SETBIT创建一个 Redis 支持的 Bloom Filter 。

#### 2. List
1. List 是一个简单的 String 列表，按插入顺序排序。可以通过 LPUSH 命令在头部插入一个元素，用 RPUSH 在尾部插入一个元素。

1. 对空键执行 PUSH 操作时，将创建一个新列表。同样，如果列表操作将清空列表，则从键空间中删除键。

1. 列表的最大长度为 2^32 - 1 个元素（4294967295，每个列表超过 40 亿个元素）。

1. 插入删除头部、尾部数据时 O(1) 操作，范问中间的数据时 O(N) 操作。

1. 应用场景：
   * 在社交网络中为时间线建模，使用 LPUSH 以在用户时间线中添加新元素，并使用 LRANGE 以检索一些最近插入的项目。
   * 您可以将 LPUSH 与 LTRIM 一起使用来创建一个永远不会超过给定元素数量的列表，而只记住最新的 N 个元素。
   * 列表可以用作消息传递原语，例如，参见著名的 Resque Ruby 库，用于创建后台作业。
   * 你可以用列表做更多的事情，这种数据类型支持许多命令，包括像 BLPOP 这样的阻塞命令。

#### 3. Set
1. Redis Set 是 String 的无序集合。可以在 O(1) 中添加、删除和测试成员的存在。

1. Redis Set 具有不允许有重复元素，支持许多服务器端命令来从现有集合开始计算集合，因此您可以在很短的时间内进行集合的并集、交集、差集。

1. 集合中的最大成员数为 2^32 - 1（4294967295，每组超过 40 亿个成员）。

1. 应用场景：

   * 您可以使用 Redis Sets 跟踪独特的事物。想知道访问给定博客文章的所有唯一 IP 地址吗？
   * Redis Set 可以很好地表示关系。您可以使用 Redis 创建一个标记系统，使用 Set 来表示每个标记。然后，您可以使用SADD命令将具有给定标签的所有对象的所有 ID 添加到表示该特定标签的集合中。您是否希望所有对象的所有 ID 同时具有三个不同的标签？只需使用 SINTER。
   * 您可以使用 Sets 使用 SPOP 或 SRANDMEMBER 命令随机提取元素。

#### 4. Hash
1. Redis Hash 是字符串字段和字符串值之间的映射，因此它们是表示对象的完美数据类型（例如，具有多个字段的用户，如姓名、姓氏、年龄等）

1. 具有几个字段（其中很少意味着最多一百个左右）的哈希以占用非常少的空间的方式存储，因此您可以在一个小型 Redis 实例中存储数百万个对象。

1. 虽然哈希主要用于表示对象，但它们能够存储许多元素，因此您也可以将哈希用于许多其他任务。

1. 每个散列最多可以存储 2^32 - 1 个字段值对（超过 40 亿个）。

#### 5. ZSet
Redis Sorted Sets 与 Redis Sets 类似，是字符串的非重复集合。不同之处在于，Sorted Set 的每个成员都与一个分数相关联，用于保持 Sorted Set 的顺序，从最小到最大的分数。虽然成员是唯一的，但分数可能会重复。

使用排序集，可以以非常快速的方式添加、删除或更新元素（时间与元素数量的对数成正比）。由于元素是按顺序存储的，而不是事后排序的，因此您还可以通过分数或排名（位置）以非常快速的方式获取范围。访问 Sorted Set 的中间也非常快，因此您可以将 Sorted Sets 用作非重复元素的智能列表，您可以在其中快速访问所需的一切：元素按顺序、快速存在测试、快速访问中间元素！

简而言之，使用 Sorted Sets，您可以执行许多性能出色的任务，而这些任务在其他类型的数据库中很难建模。

使用排序集，您可以：

1. 在大型在线游戏中建立排行榜，每次提交新分数时，您都使用ZADD对其进行更新。您可以使用ZRANGE轻松检索排名靠前的用户，还可以在给定用户名的情况下使用ZRANK返回其在列表中的排名。将 ZRANK 和 ZRANGE 一起使用，您可以向用户显示与给定用户相似的分数。一切都很快。
1. 排序集通常用于索引存储在 Redis 中的数据。例如，如果您有许多代表用户的散列，您可以使用一个排序集，其中成员以用户的年龄作为分数，用户的 ID 作为值。因此，使用ZRANGEBYSCORE检索具有给定年龄范围的所有用户将是简单而快速的。
排序集是更高级的 Redis 数据类型之一，因此请花一些时间查看排序集命令的完整列表，以了解您可以使用 Redis 做什么！此外，您可能还想阅读Redis 数据类型简介。

#### 6. 其他类型
* Bitmaps（位图）
* HyperLogLogs（基数统计）
* Streams（流）
* Geospatial indexes（地理空间索引）

## 原理篇
### 为什么要用 Redis，Redis 有什么优点？
1. **读写性能优异**：Redis 能读的速度是 110000 次/s，写的速度是 81000 次/s

1. **数据类型丰富**：Redis 支持二进制案例的 Strings，Lists，Hashes，Sets 及 Ordered Sets 数据类型操作。

1. **原子性**：Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行。

1. **丰富的特性**：Redis 支持 publish/subscribe, 通知, key 过期等特性。

1. **持久化**：Redis支持RDB, AOF等持久化方式

1. **发布订阅**：Redis支持发布/订阅模式

1. **分布式**：Redis Cluster

### Redis 为什么是单线程？Redis 为什么这么快？

1. Redis 完全基于内存，绝大部分请求是纯粹的**内存操作**，非常快速

1. **数据结构简单**，对数据操作也简单，Redis 中的数据结构是专门进行设计的
1. 采用**单线程**模型，避免了不必要的上下文切换和竞争条件，也不存在多线程或者多线程切换而消耗 CPU，不用考虑各种锁的问题，不存在加锁，释放锁的操作，没有因为可能出现死锁而导致性能消耗
1. 使用了 **IO 多路复用**模型，非阻塞 IO
1. 使用**底层模型不同**，它们之间底层实现方式及与客户端之间的通信的应用协议不一样，Redis 直接构建了自己的 VM 机制，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求

### Redis 持久化有哪些？
1. **RDB**（生产环境一般用 RDB）：周期性快照持久化、用作灾难备份、恢复速度快，不保证数据完整性和一致性；备份时占用内存；

2. **AOF**：把每条写入作为日志追加写入到日志文件中，类似 binlog；优点：保证数据完整性；缺点：文件大、恢复速度慢

3. **混合模式**：RDB + AOF，恢复时 redis 启动时先检查 AOF 文件是否存在，不存在则尝试加载 RDB 文件，灾难恢复时可以考虑先恢复 RDB 文件（更快恢复数据），然后再通过 AOF 做数据补全。

### Redis 淘汰策略有哪些？

1. 被动删除：只有被 get 到发现过期的时候，删除并返回 NIL，属于惰性删除
2. 主动删除：100ms 运行一次，随机删除持续 25ms，类似 Cron
3. 内存使用超过 maxmemory，触发主动清理策略

   * volatile-lru（默认）：从设置过期数据集里查找最近最少使用
   * volatile-ttl：从设置过期的数据集里面优先删除剩余时间短的Key
   * volatile-random：从设置过期的数据集里面任意选择数据淘汰
   * volatile-lfu：从过期的数据集里删除 最近不常使用 的数据淘汰
   * allkeys-lru
   * allkeys-lfu
   * allkeys-random：数据被使用频次最少的,优先被淘汰
   * no-enviction

> 如果不设置 maxmemory，Redis 将一直使用内存，直到触发操作系统的 OOM-KILLER

### Redis 有哪些数据类型？底层实现方式？
Redis 有 5 种基本数据类型：**String、Hash、List、Set、ZSet**；和 5 种不常用的类型：Pubsub（发布订阅）、Bitmap（位图）、GEO（地理位置）、Stream（流）、Hyperloglog（基数统计）。

数据量较小和数据量大的时候底层实现往往不一样，主要关注大数据量结构。

1. **String**：数字的时候是 int，长字符串（长度大于 39 字节）是 raw，短字符串（长度小于 39 字节）是 embstr。

   embstr 和 raw 都是由 SDS 动态字符串构成，唯一区别是：分配内存的时候，raw 是 redisobject 和 sds 各分配一块内存，而 embstr 是在一块儿内存中。

1. **Hash**：列表对象所有字符串元素长度都小于 64 个字节，且元素数量小于 512 时通过 ziplist 实现，否则通过哈希表实现。

1. **List**：列表对象所有字符串元素长度都小于 64 个字节，且元素数量小于 512 时通过 ziplist 实现，否则通过双向链表实现。

1. **Set**：所有元素都是整数，且元素数量小于 512 时，通过 inset 实现，否则通过哈希表实现。

1. **Zset**：元素数量小于 128，且所有元素长度小于 64 时，通过 ziplist 实现，否则通过跳表实现。

   是一个有序集合，可以作用于排序场景、排行榜场景

### Redis 底层数据结构有哪些？

1. **SDS**【动态字符串】：redis 中所有场景中出现的字符串，基本都是由 SDS 来实现的。包括：所有非数字的 key、字符串数据类型的值、非字符串数据类型中的“字符串值”。

1. **ZipList**【压缩列表】：此数据结构是为了节约内存而开发的。和数组类似，它是由连续的内存块组成的，由于内存是连续的，就减少了很多内存碎片和指针的内存占用，进而节约了内存。

1. **IntSet**【整数集】：整数集合是集合键的底层实现方式之一。

1. **ZSkipList**【跳表】：通过对链表建立多层索引的方式提交查询效率的数据结构。
   Redis 的跳跃表由 zskiplistNode 和 skiplist 两个结构定义，其中 zskiplistNode 结构用于表示跳跃表节点，而 zskiplist 结构则用于保存跳跃表节点的相关信息，比如节点的数量，以及指向表头节点和表尾节点的指针等等。

1. **Dict**【字典 / 哈希表】：拉链法实现。

1. **QuickList**【快表】

## 实践篇

### 什么是缓存击穿？缓存穿透？缓存雪崩？
1. **缓存击穿**：指热点 key 在某个时间点过期的时候，而恰好在这个时间点对这个 Key 有大量的并发请求过来，从而大量的请求打到 db。

   解决方法：采用分布式锁，只有拿到锁的第一个线程去请求数据库，然后插入缓存。

1. **缓存穿透**：指查询一个一定不存在的数据，由于缓存是不命中时需要从数据库查询，查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到数据库去查询，进而给数据库带来压力。

   解决方法：给相应的Key设置一个 Null 值，放在缓存中；BloomFilter 预先判断；

1. **缓存雪崩**：指缓存中数据大批量到过期时间或者 Redis 宕机，而查询数据量巨大，引起数据库压力过大甚至 down 机。

   解决方法：给失效时间加上相对的随机数；保证 Redis 的高可用；

### Redis 有哪些常见用法和应用场景？

1. 缓存

1. 分布式锁
1. 分布式 id 生成
1. 特殊场景
   * zset 排行榜，排序
   * bitmap 用户签到，在线状态
   * geo 地理位置，附近的人
   * stream 类似 kafka 的消息流
   * hyperloglog 每日访问 ip 数统计

1. 分布式限流

1. Session

1. 发布与订阅

```
## 基于频道的实现
# 订阅多个频道
subscribe music movie
# 向music频道发布消息：love
publish music love
# 查看被订阅的频道列表
punsub channels [channel_name]
# 查看指定频道的订阅数
pubsub numsub channel_name [channel_name]

## 基于模式的实现
# 订阅一个或者多个模式
psubscribe pattern-1 pattern-2
# 取消模式的订阅
punsubscribe pattern-1 pattern-1
# 向频道 channel-1 发送消息 message
publish channel-1 message
# 查询当前服务器被订阅模式的数量
pubsub numpat
```

### Redis 常用客户端有哪些？
* lettuce：SpringBoot 默认，基于 Netty 的事件驱动模型
* jedis：老牌的客户端，使用 commons-pool 来完成线程池开发
* redisson：非常丰富的分布式数据结构，包括锁，分布式 Map 等。大量使用 Lua 脚本️

### Redis 使用规范
* 使用连接池，不要频繁创建关闭客户端连接

* 消息大小限制 消息体在 10kb 以下，可以使用 snappy、msgpack 等压缩
* 避免大 key 和 hot key
* 不使用 O(n) 指令
* 不使用不带范围的 Zrange 指令
* 不使用 database（容易覆盖数据）
* 不使用高级数据结构（使用基本的 5 种）
* 不使用事务操作
* 禁止长时间 monitor（实时打印出 Redis 服务器接收到的命令）
* 禁止使用 flushall（清空整个 Redis 服务器的数据）、flushdb（清空当前数据库中的所有 key）
* 注意使用 del 命令，List/Hash/Set/ZSet 复杂度是 O(n)，list 可以用`lpop 或者 rpop`，Hash/Set/ZSet 可以用`hdel/srem/zrem`

> O(n) 指令：keys *、hgetall、smembers、sunion；要避免使用 O(n) 指令，最好使用 RENAME 指令给屏蔽掉

### Redis 有哪些集群方案？

数据同步分为全量同步和部分同步，新创建从机进行一次全量同步，之后进行增量同步

1. 单机 & 单机多实例

1. **主从模式**：一主多从，master 出问题，选取一个 slave 顶上，主从切换比较麻烦，可以通过 keepalived 实现

1. **哨兵模式**：使用额外的进程替换 keepalived 判断 redis 存活性，哨兵数量至少需要 3 个。这种模式需要增加哨兵监测节点状况，然后通过指令完成主从切换或节点状态改变。

1. **Redis Cluster**：官方集群方案，官方推荐，适合多主（分片）多从（副本）

1. **Codis**

### Redis 问题排查
* monitor 指令：回显所有执行的指令。可以使用 grep 配合过滤
* keyspace-events：订阅某些 Key 的事件。比如，删除某条数据的事件，底层实现基于 pubsub
* slow log：顾名思义，满查询，非常有用
* --bigkeys 启动参数：Redis 大 Key 健康检查。使用的是 scan 的方式执行， 不用担心阻塞
* memory：usage key、memory stats 指令
* info 指令：关注 instantaneous_ops_per_sec、used_memory_human、connected_clients
* redis-rdb-tools：rdb 线下分析

### Redis 热点数据问题
#### 1. 如何处理热 key 出现造成集群访问量倾斜
**出现原因**：一段时间内，某些 key 访问量远高于其他 key，导致大部分流量在 proxy 分片后集中访问到某一个 redis 实例上。

解决方案：1. 使用本地缓存：需要找出热点 key，缓存不能过大、需要保证和 redis 数据的一致性；2. 利用分片算法的特性，对 key 进行打散处理。

#### 2. 大 key 造成的集群数据量倾斜
数据量大的 key，数据远大于其他 key，导致分片后存储这个 key 的节点内存使用远大于其他节点，内存不足拖累整个集群。可以通过对大 key 进行拆分的方法解决。

#### 3. 热 key、大 key 发现
1. 事前预判
1. 事中监控：应用端上报；代理层上报；redis 命令；抓包统计；

### Redis 常用命令
```
# 远程连接
./redis-cli -h 192.168.137.151 -p 6379
auth nRDPkyJWKgRNo1j4M48=

# 设置key-value及过期时间
set name pleuvoir ex 10
# 查看过期时间
ttl name
# 如果没有则设置，否则不操作（分布式锁常用）
setnx name pleuvoir

# 将一组Redis命令进行组装，通过一次RTT（命令执行往返时间）传输给Redis，再将这组Redis命令的执行结果按顺序返回给客户端
Pipeline

# 如果没有则设置，否则不操作（分布式锁常用）
setnx [key] [value]

# 远程连接；auth [pass]：认证
./redis-cli -h [ip] -p 6379

# 自增
incr age
# 自减
decr age

# Lua脚本实现原子操作
/**
 * lua-script：Lua 脚本内容
 * numkeys：表示的是 Lua 脚本中需要用到多少个 key，如果没用到则写 0
 * key [key ...]：将 key 作为参数按顺序传递到 Lua 脚本，numkeys 是 0 时则可省略
 * arg：Lua 脚本中用到的参数，如果没有可省略
 */
eval lua-script numkeys key [key ...] arg [arg ...]

/**
 * command：Redis 中的命令，如 set、get 等
 * key：操作 Redis 中的 key 值，相当于我们调用方法时的形参
 * param：代表参数，相当于我们调用方法时的实参
 */
redis.call(command, key [key ...] argv [argv…])

# 摘要：判断一个摘要是否存在。0 表示不存在，1 表示存在。
script exists
# 清除所有 Lua 脚本缓存
script flush
# 执行Lua 脚本文件
redis-cli --eval test.lua 1 age , 18
```

## 常见问题
### Redis 扩容机制？底层 Dict 如何实现？

## 参考
* [Redis，就是这么朴实无华](http://xjjdog.cn/arch/16065216929591.html)

* [从应用到底层 36张图带你进入Redis世界](https://juejin.cn/post/6906680666214105102#heading-32)

* [要想用活Redis，Lua脚本是绕不过去的坎](https://juejin.cn/post/6926779957540732935)

* [图解redis五种数据结构底层实现(动图哦)
](https://juejin.cn/post/6844904008591605767#heading-4)

* [万字总结redis知识点](https://juejin.cn/post/6999986774114058270#comment)

* [Redis 底层数据结构 dict（hashtable）的实现机制](https://juejin.cn/post/7047665450632609805)