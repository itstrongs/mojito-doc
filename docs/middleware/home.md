# MySQL

## MySQL基础
### MySQL 事务

### 什么是事务？事务的基本特性有哪些？如何保证？
满足 ACID 特性的一组操作叫做事务。

1. **原子性**（Atomicity）：原子性是指事务的原子性操作，对数据的修改要么全部执行成功，要么全部失败。由 undo log 日志保证，它记录了需要回滚的日志信息，事务回滚时撤销已经执行成功的 sql；

1. **一致性**（Consistency）：一致性是指执行事务前后的状态要一致，可以理解为数据一致性。一般由代码层面来保证；

1. **隔离性**（Isolation）：隔离性侧重指事务之间相互隔离，不受影响，这个与事务设置的隔离级别有密切的关系。由 MVCC 来保证。

1. **持久性**（Durability）：持久性则是指在一个事务提交后，这个事务的状态会被持久化到数据库中，也就是事务提交，对数据的新增、更新将会持久化到书库中。由内存 + redo log 来保证，mysql 修改数据同时在内存和 redo log 记录这次操作，事务提交的时候通过 redo log 刷盘，宕机的时候可以从 redo log 恢复。

### 事务有哪些隔离级别？
1. **读未提交**【READ UNCOMMITTED】：其他事务可以读取当前事务未提交的数据。当当前事务更新数据后，其他事务之前读到的就是脏数据，造成**脏读**。

1. **读已提交**【READ COMMITTED】：一个事务只能读取已经提交的事务所做的修改。多次去读可能发生其他事务修改提交后的值，造成多次读取不一致，造成**不可重复读**。

1. **可重复读（默认）**【REPEATABLE READ】：保证在同一个事务中多次读取同样数据的结果是一样的。通常通过读锁来保证，但是读锁只能锁住指定行数据，当事务读取某个范围后，其他事务在这个范围插入数据，再读的结果就跟之前读的不一样了，导致**幻读**。通常可以通过间隙锁保证。

1. **串行化**【SERIALIZABLE】：强制事务串行执行。

### 什么是 MVVC ？如何实现？
全称 Multi-Version Concurrency Control（**多版本并发控制**），一种并发控制的方法，通过**版本链**，实现多版本，可并发读-写，写-读。

在 InnoDB 引擎中就是指在已提交读（READ COMMITTD）和可重复读（REPEATABLE READ）这两种隔离级别下的事务对于 SELECT 操作会访问版本链中的记录的过程。

这就使得别的事务可以修改这条记录，反正每次修改都会在版本链中记录。SELECT 可以去版本链中拿记录，这就实现了读-写，写-读的并发执行，提升了系统的性能。

实现方式：主要通过**隐式字段、undo log、ReadView** 实现。

### MySQL 索引

### 什么是 B+ 树？为什么用 B+ 树实现索引？B+ 树索引有哪些特点？
* **什么是 B+ 树**：B+ 树是基于 B 树和叶子节点顺序访问指针进行实现，它具有 B 树的平衡性，并且通过顺序访问指针来提高区间查询的性能。

* **为什么用 B+ 树**：因为索引的主要目的是加快查找，包括范围查找。

   B 树通过降低树的高低（很低的常量级高度）减少磁盘 IO 次数（IO 耗时 = 寻道时间 + 旋转延迟 + 传输，树的高度决定了 IO 次数）来加快了查询速度，B+ 树通过叶子节点顺序指针很好的支持了范围查找。

* **B+ 树索引有哪些特点：**
   1. 一次 IO 一般读取一页的数据【预读】，一页一般为 4k 或 8k【跟操作系统有关】
   1. B+ 树由根节点、非叶子节点和叶子节点组成，非叶子节点不存储数据，只存储指针地址，数据都存储在叶子节点，叶子节点通过双向链表连接

### 什么是聚簇索引和非聚簇索引？
InnoDB 中，表都以索引的形式存在，每张表必定有主键索引，根据主键构建一颗 B+ 树，所有数据都存在主键索引叶子节点，索引和数据在一起所以叫聚簇索引，索引和数据不在一起所以叫非聚簇索引。

* 聚簇索引 = 聚集索引 = 主键索引 = 一级索引
* 非聚簇索引 = 非聚集索引 = 普通索引 = 二级索引 = 辅助索引

### 什么是索引覆盖和回表？
索引覆盖指的是在一次查询中，如果一个索引包含或者说覆盖所有需要查询的字段的值，我们就称之为索引覆盖，而不再需要回表查询。回表就是查询普通索引获取到主键索引 id 之后再去主键索引获取数据。

可以通过 explain 查看是否是覆盖索引：Extra 是否是 Using index。

### 索引高度计算
B+ 树节点由磁盘块组成，每个磁盘块由数据项和指针组成。非叶子节点数据项存储的是指引搜索方向的数据项，叶子结点数据项存储的是真实的数据。

假设索引高度为 h，表的数据为 N，每个磁盘块的可以存储的索引数量为 m，那 h = logmN = lgN / lgM

比如有 1 亿条数据，每个磁盘快可以存储 100 个索引，那高度为：log100 10^8 = 4

每个磁盘块可以存储的数量 = 磁盘块大小 / (数据项大小 + 指针大小)

磁盘块大小一般为操作系统页的整数倍：4k 或 16k，数据项大小由列类型决定，比如 bigint 是 8 字节，假设平均指针大小是 8 字节，则存储数量为 4k / (8 + 8) = 256

### 大表如何添加索引
表加索引会锁表，可能导致生产事故。

### MySQL 日志系统

日志是 MySQL 数据库的重要组成部分，记录着数据库运行期间各种状态信息。MySQL 日志主要包括：错误日志、查询日志、慢查询日志、事务日志（redo log、undo log）、二进制日志（binlog）。

### binlog

#### 1. 什么是 binlog ？
binlog 用于记录数据库执行的**写入性操作**（不包括查询）信息，以**二进制**的形式保存在磁盘中。binlog 是 MySQL 的**逻辑日志**，并且由 Server 层进行记录，使用任何存储引擎的 mysql 数据库都会记录 binlog 日志。

binlog 是通过**追加**的方式进行写入的，可以通过 max_binlog_size 参数设置每个 binlog 文件的大小，当文件大小达到给定值之后，会生成新的文件来保存日志。

> **逻辑日志**：可以简单理解为记录的就是 sql 语句。<br>
> **物理日志**：因为 MySQL 数据最终是保存在数据页中的，物理日志记录的就是数据页变更。

#### 2. binlog 主要作用有哪些？
1. **主从复制**：在 Master 端开启 binlog，然后将 binlog 发送到各个 Slave 端，Slave 端重放 binlog 从而达到主从数据一致。

1. **数据恢复**：通过使用 MySQL binlog 工具来恢复数据。

#### 3. binlog 刷盘时机是什么？
对于 InnoDB 存储引擎而言，只有在事务提交时才会记录 biglog，此时记录还在内存中，MySQL 通过 sync_binlog 参数控制 biglog 的刷盘时机，取值范围是 0-N：

* 0：不去强制要求，由系统自行判断何时写入磁盘；
* 1：每次 commit 的时候都要将 binlog 写入磁盘；
* N：每 N 个事务，才会将 binlog 写入磁盘。

从上面可以看出，sync_binlog 最安全的是设置是 1，这也是 MySQL 5.7.7 之后版本的默认值。但是设置一个大一些的值可以提升数据库性能，因此实际情况下也可以将值适当调大，牺牲一定的一致性来获取更好的性能。

#### 4. binlog 日志格式有哪些？
binlog 日志有三种格式，分别为 STATMENT、ROW 和 MIXED。在 MySQL 5.7.7 之前，默认的格式是 STATEMENT，MySQL 5.7.7 之后，默认值是 ROW。日志格式通过 binlog-format 指定：

1. **STATMENT**：基于 SQL 语句的复制（statement-based replication，SBR），每一条会修改数据的 sql 语句会记录到 binlog 中。
   * 优点：不需要记录每一行的变化，减少了 binlog 日志量，节约了 IO, 从而提高了性能；
   * 缺点：在某些情况下会导致主从数据不一致，比如执行 sysdate()、slepp() 等。

1. **ROW**：基于行的复制（row-based replication，RBR），不记录每条 sql 语句的上下文信息，仅需记录哪条数据被修改了。
   * 优点：不会出现某些特定情况下的存储过程、或 function、或 trigger 的调用和触发无法被正确复制的问题；
   * 缺点：会产生大量的日志，尤其是 alter table 的时候会让日志暴涨；

1. **MIXED**：基于 STATMENT 和 ROW 两种模式的混合复制（mixed-based replication, MBR），一般的复制使用 STATEMENT 模式保存 binlog，对于 STATEMENT 模式无法复制的操作使用 ROW 模式保存 binlog。

### redo log

#### 1. redo log 基本概念

redo log 主要是用来实现事务持久性的，记录了事务对数据页做了哪些修改。

redo log 包括两部分：一个是内存中的**日志缓冲**（redo log buffer），另一个是**磁盘**上的日志文件（redo log file）。MySQL 每执行一条 DML 语句，先将记录写入 redo log buffer，后续某个时间点再一次性将多个操作记录写到 redo log file。这种先写日志，再写磁盘的技术就是 MySQL 里经常说到的 WAL（Write-Ahead Logging）技术。

在计算机操作系统中，用户空间（user space）下的缓冲区数据一般情况下是无法直接写入磁盘的，中间必须经过操作系统内核空间（kernel space）缓冲区（OS Buffer）。因此，redo log buffer 写入 redo log file 实际上是先写入 OS Buffer，然后再通过系统调用 fsync() 将其刷到 redo log file 中。

#### 2. redo log buffer 写入磁盘时机？

MySQL 支持三种将 redo log buffer 写入 redo log file 的时机，可以通过 innodb_flush_log_at_trx_commit 参数配置，各参数值含义如下：

* **0（延迟写）**：事务提交时不会将 redo log buffer 中日志写入到 os buffer，而是每秒写入 os buffer 并调用 fsync() 写入到 redo log file 中。也就是说设置为 0 时是（大约）每秒刷新写入到磁盘中的，当系统崩溃，会丢失 1 秒钟的数据。

* **1（实时写，实时刷）**：事务每次提交都会将 redo log buffer 中的日志写入 os buffer 并调用 fsync() 刷到 redo log file 中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO 的性能较差。
* **2（实时写，延迟刷）**：每次提交都仅写入到 os buffer，然后是每秒调用 fsync() 将 os buffer 中的日志写入到 redo log file。

### undo log
记录的是数据操作前的样子，保证了事务的持久性。

### MySQL 数据库锁
### 共享锁（读锁）、排它锁（写锁）
* 读锁只能读不能写，加锁语句：lock in share mode
* 写锁是排他的，加锁语句：for update，其他事务要等当前事务提交之后才能执行，默认行级锁，基于索引，没有索引会用表级锁
* 表锁、行锁、间隙锁、临键锁都是排它锁

### 行锁（记录锁）、页级锁、表锁
* **表锁**：开销小，并发差，修改表结构等触发；**行锁**：锁定每一行数据，开销大，并发强；**页级锁**：介于两者之间

* 意向锁是表级锁，其他大部分是行级锁

### 间隙锁（Gap Locks）
* 间隙锁是封锁索引记录之间的空白间隔，是为了解决幻读问题
* 只有在事务隔离级别 RR 中才会产生
* 产生情况：普通索引、多列唯一索引、唯一索引锁定多行记录
* 只使用行锁，不会产生间隙锁
* 锁范围：假如唯一索引为：1、5、7、11，则间隙锁范围：(-infinity, 1]、(1, 5]、(5, 7]、(7, 11]、(11, +infinity]
* 唯一索引：查询某一条记录的时候，记录存在不会产生间隙锁，不存在会产生，锁范围为查询记录所在区间，比如查询不存在的 3，则锁住 (1, 5]；查询某一范围会产生间隙锁；
* 普通索引：在普通索引列上，不管是何种查询，只要加锁，都会产生间隙锁，这跟唯一索引不一样；在普通索引跟唯一索引中，数据间隙的分析，数据行是优先根据普通索引排序，再根据唯一索引排序。

### 临键锁（Next-Key lock）
* 行锁与间隙锁的组合
* 加锁的基本单位是 next-key lock

### 意向锁
* **意向共享锁**（IS锁）、**意向排它锁**（IX锁）

* 意向锁的目的就是表明有事务正在或者将要锁住某个表中的行，实际使用较少
* 前开后闭区间

### MySQL 常见问题

### MySQL 执行流程与优化器
1. SQL 通过解析器进行**词法、语法解析**，得到一颗语法树
2. 基于语法树做一些**准备工作**，如语义检查、表名库名补全等
3. 利用关系代数对 SQL **等价转换**，化简优化，如否定消除、等值常量传递、常量表达式评估、 外连接转内连接、子查询转半连接等
4. 主要**分析**条件表达都可以走哪些索引、有哪些访问方式(如全表扫、家引范国扫）、预估条件行数、常量表检测等
5. 根据代价模型**选出**使用的紫引、表引用的最优访问方式、Join 的连接顺序等
6. 进行**额外的优化**，比如使用素引条件下推减少10操作，最后输出一个执行计划
7. 调用存储引擎的接口，**运行**执行计划

### MySQL 基本组成部分？
MySQL分为三部分：`客户端`、`服务端`和`存储引擎`（存储引擎可以切换），客户端发送请求到服务端，服务端通过连接器、缓存、分析器、优化器、执行器后调用存储引擎接口获取数据返回。

首先连接数据库，执行 SQL 语句，通过词法分析语法分析，通过优化分析器分析要不要走索引，确定最快的查询方式，然后计算扫描区间，获取扫描数据索引后回表获取对应的行数据。

**连接器**：建立连接、验证权限、维持管理连接，支持短连接和长连接（默认8小时，通过连接池使用）；

**缓存**：默认关闭，8.0后移除；

**InnoDB存储引擎**：内存架构、磁盘架构

### 从准备更新一条数据到事务的提交的流程描述？
1. 首先执行器根据 MySQL 的执行计划来查询数据，先是从缓存池中查询数据，如果没有就会去数据库中查询，如果查询到了就将其放到缓存池中

1. 在数据被缓存到缓存池的同时，会写入 undo log 日志文件
1. 更新的动作是在 BufferPool 中完成的，同时会将更新后的数据添加到 redo log buffer 中
1. 完成以后就可以提交事务，在提交的同时会做以下三件事
   1. 将 redo log buffer 中的数据刷入到 redo log 文件中
   1. 将本次操作记录写入到 bin log 文件中
   1. 将 bin log 文件名字和更新内容在 bin log 中的位置记录到 redo log 中，同时在 redo log 最后添加 commit 标记

### myisam 和 innodb 的区别？
myisam 引擎是 5.1 版本之前的默认引擎，支持全文检索、压缩、空间函数等，但是不支持事务和行级锁，所以一般用于有大量查询少量插入的场景来使用，而且 myisam 不支持外键，并且索引和数据是分开存储的。

innodb 是基于 B+ Tree 索引建立的，和 myisam 相反它支持事务、外键，并且通过 MVCC 来支持高并发，索引和数据存储在一起。

### MySQL 分区如何使用？

[MySQL分区表最佳实践](https://juejin.cn/post/6844904174241447949)

### MySQL 连表查询

MySQL 连表查询只有 3 种：内连接、外连接、自然连接。内连接又叫交叉连接，外连接分为左连接和右连接。

我们知道，连接查询其实就是拿表 A 的记录去表 B 中匹配，查询的表称为驱动表，被查询的表称为被驱动表。

连接查询是每次从表 A 拿一条记录去表 B 匹配，而表 B 匹配的效率是固定的（不管有没有做索引），所以你可以看到连接查询的效率取决于表 A（驱动表）的记录数（行数），这也是为什么人们常说要用“**小表驱动大表**”的原因。

### 内连接
内连接的关键字是`INNER JOIN`，`INNER JOIN`等于`CROSS JOIN`等于`JOIN`。

内连接分为两种情况：
1. 没有筛选条件，此时得到的结果是两张表的笛卡尔积；
1. 有 ON 和 WHERE 筛选条件，此时得到的结果是两张表的交集，对于内连接，ON 和 WHERE 是等价的，但是对于外连接则不是。

### 外连接
之所以有外连接，是因为内连接在一些场景下并不能满足我们的要求，内连接的原理是从每次 A 表取一条记录去 B 表中匹配，匹配成功则保留，匹配失败则放弃，直到 A 表的记录遍历完。

但有时候，我们需要保留匹配失败的记录，比如我们 A 表是学生表，B 表是分数表，当我们拿学生去 B 表查分数时，如果没找到，我们是比较希望保留该学生的信息的，所以就出现了外连接。

MySQL 并不支持`OUTER JOIN`，但是左外连接和右外连接是支持的，他们的区别就在于要保留哪份表的数据。

* **左连接**：左连接的关键字是`LEFT JOIN`，左连接其实就是两个表的交集 + 左表剩下的数据 ，当然这是在没其他过滤条件的情况下。

* **右连接**：右连接的关键字是`RIGHT JOIN`，从上图可以得到（右边的图），右连接其实就是两个表的交集 + 右表剩下的数据 ，当然这是在没其他过滤条件的情况下。

ON 子句中的过滤条件对于内连接和外连接是不同的，对于内连接，ON 和 WHERE 的作用是一致的，因为匹配不到的都会过滤，所以你可以看到内连接并不强制需要 ON 关键字；但是对于外连接，ON 决定匹配不到的是否要过滤，所以你可以看到外连接是强制需要 ON 关键字的。

### 自然连接
自然连接的关键字是`NATURAL JOIN`，自然连接与内连接的区别在于，自然连接是一种自动寻找连接条件的连接查询，即 MySQL 会自动寻找相同的字段名作为连接条件，当没有找到连接条件，就变成内连接（没有筛选条件的情况）。


### 参考
* [听说有一个最左原则？这回终于讲清楚了MySQL执行查询时联合索引用几个列的问题](https://juejin.im/post/6844904169619488775)

* [MySQL性能优化之参数配置](https://www.cnblogs.com/angryprogrammer/p/6667741.html)
* [一个慢SQL引起的惨案](https://juejin.im/post/5e9dafd06fb9a03c585c1050)
* [很用心的为你写了 9 道 MySQL 面试题](https://juejin.im/post/5e9a65326fb9a03c5d5c90e6)
* [不会看 Explain执行计划，劝你简历别写熟悉 SQL优化](https://juejin.im/post/5ec4e4a5e51d45786973b357)
* [为什么要避免大事务以及大事务如何解决？](https://segmentfault.com/a/1190000023273980)
* [MySQL的锁机制 - 记录锁、间隙锁、临键锁](https://zhuanlan.zhihu.com/p/48269420)
* [MySQL索引原理及慢查询优化](https://tech.meituan.com/2014/06/30/mysql-index.html)
* [这一次，彻底读懂Mysql执行计划](https://juejin.cn/post/6844903545607553037)

* [pt解析慢查询日志](https://blog.csdn.net/jiedaodezhuti/article/details/106661145)

* [MySQL为什么取消了Query Cache?](https://cloud.tencent.com/developer/article/1693427)
* [必须了解的mysql三大日志-binlog、redo log和undo log](https://juejin.cn/post/6860252224930070536#heading-9)

## MySQL 常用命令
### MySQL 连接数
```bash
# 静态查看
SHOW PROCESSLIST;  
SHOW FULL PROCESSLIST;  
SHOW VARIABLES LIKE '%max_connections%';  
SHOW STATUS LIKE '%Connection%'; 
# 重新设置最大连接数
set global max_connections=1000;

# 实时查看
show status like 'Threads%';

+-------------------+-------+  
| Variable_name     | Value |  
+-------------------+-------+  
| Threads_cached    | 58    |  
| Threads_connected | 57    |   # 这个数值指的是打开的连接数，Threads_connected 跟show processlist结果相同，表示当前连接数。
| Threads_created   | 3676  |   # threads_created表示创建过的线程数，如果发现threads_created值过大的话，表明mysql服务器一直在创建线程，这也是比较耗资源
| Threads_running   | 4     |   # 这个数值指的是激活的连接数，这个数值一般远低于connected数值，准确的来说，Threads_running是代表当前并发数
+-------------------+-------+  
```

### 基本命令
```bash
# 查看正在运行中的命令
select * from information_schema.`PROCESSLIST` where info is not null ORDER BY time desc;

# 查看慢SQL是否开启
show variables like "slow_query_log%";
# 查看慢查询设定的阈值 单位:秒
show variables like "long_query_time";

# 查看事务
select * from information_schema.innodb_trx;
# 查询执行时间超过10秒的事务
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>10
# 查看锁
select * from information_schema.INNODB_LOCKS;
# 查看锁等待
select * from information_schema.INNODB_LOCK_WAITS;
# 查看所有进程
select * from processlist;

# 查询表结构
show create table table_name

# 进入命令行
mysql -u root -p

SHOW BINLOG EVENTS\G
-- 查询binlog文件名列表
SHOW BINARY LOGS;
-- 查询指定文件的binlog
SHOW BINLOG EVENTS IN 'master-bin.000002'\G

# mysql远程连接
mysql -h 175.24.93.154 -P 3306 -u root -plfq12345
```