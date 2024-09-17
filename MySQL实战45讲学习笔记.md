[toc]

极客时间 林晓斌 mysql实战45讲

# MySQL实战45讲-学习笔记

## 01 基础架构：一条SQL查询语句是如何执行的？

### mysql逻辑架构

> ![mysql逻辑架构图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/01/1.png)
>
> MySQL逻辑架构图

mysql大体分为server层和存储引擎层。

所有跨存储引擎的功能在server层实现，比如存储过程、触发器、视图。

存储引擎层负责数据存取，提供读写接口。InnoDB是mysql5.6版本后的默认存储引擎。

#### 连接器

连接命令：

`mysql -h [ip] -P [port] -u [user] -p`

负责跟客户端建立连接、获取权限、维持和管理连接。

当用户名密码认证通过后，连接器会到权限表里查询用户拥有的权限，之后这个连接里的权限判断逻辑都依赖于此。意味着连接成功建立后，即使对这个用户的权限做了修改，也不影响已经存在的连接的权限，只有新建的连接才会使用新的权限设置。

数据库里长连接指连接成功后，如果客户端持续有请求则一直用同一个连接；短连接指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。

尽量使用长连接减少开销。mysql在执行过程中临时使用的内存管理在连接对象里，在连接断开才会释放内存。如果长连接长时间累积可能导致out of memory，表现为mysql异常重启。

解放方案为1.定期断开长连接或程序判断执行过占用大内存的查询后断开，之后查询再重连；2.mysql5.7后可以在执行占用大内存的查询后通过`mysql_reset_connection`重新初始化连接资源，不需要重连和重新验证权限，将连接恢复到刚创建时的状态。

#### 查询缓存

**经验证，mysql8.0将查询缓存功能删掉了。** 

查询缓存可能以key-value形式缓存在内存中，key是语句，value是结果。

如果语句不在查询缓存中，会执行后续阶段，执行结果会存入查询缓存。

只要有对一个表的更新，这个表上的查询缓存会全部清空，失效非常频繁。对于更新压力大的数据库命中率非常低。很长时间更新一次的静态表才适用查询缓存。

mysql可按需使用。将`query_cache_type` 设置成`DEMAND` 则默认不用查询缓存，需要时用`select SQL_CACHE ...` 显式指定。

#### 分析器

做词法分析和语法分析，判断是否符合语法规则。一般错误会提示第一个出错的位置。

#### 优化器

决定表里有多个索引时使用哪个索引；join语句时决定表的连接顺序。这个阶段确定语句的执行方案。

#### 执行器

开始执行时，先判断有没有查询权限。

如果有，就打开表继续执行，打开时根据表的引擎定义，使用引擎提供的接口。

如果字段没有索引，则调用引擎接口“取表的第一行”，然后调用接口“取下一行”循环取表的各行，每一行判断字段值，不满足则跳过，满足则将这行存入结果集中，最后把结果集返回客户端。

如果字段有索引，第一次调用“取满足条件的第一行”接口，之后循环“取满足条件的下一行”接口。

慢查询日志中`rows_examined` 字段表示这条语句执行过程中扫描了多少行，这个值在每次调用引擎获取数据行时累加。

有些场景下执行器调用一次接口，引擎内部扫描多行，因此**引擎扫描行数跟`rows_examined` 并不完全相同。**

## 02 日志系统：一条SQL更新语句如何执行

查询语句的流程，更新语句同样走一遍。在查询缓存这个阶段，更新语句清空这个表上所有缓存结果。不一样的是更新流程涉及redo log（重做日志）和binlog（归档日志）。

### redo log

InnoDB引擎特有的日志。

如果每一条更新都直接写进磁盘，需要在磁盘上找到这条记录并更新，IO和查询成本很大。

类比几十页的账本，对应磁盘；一块小黑板实时记录赊账信息，对应redo log。

WAL技术，Write-Ahead Logging，先写日志，再写磁盘。

有一条记录需要更新时，InnoDB会把记录写到redo log并更新内存，更新就算完成了，InnoDB会在适当的时候把操作记录更新到磁盘里。

InnoDB的redo log大小固定，比如配置为一组4个文件（0-3号），每个文件1GB，则总共可以记录4GB的操作。从头开始写，写到末尾又回到开头循环写。

redo log 上的两个位置： `write pos` 是当前记录的位置，一边写一边后移，写到3号末尾就回到0号开头。`checkpoint` 是当前要擦除的位置，后移并循环，擦除前先把记录更新到磁盘数据文件。

`write pos` 和 `checkpoint` 中空着的部分用来记录新的操作。当`write pos` 追上`checkpoint` ，不能再执行新的更新，需要先擦除记录，把`checkpoint` 推进。

redo log提供了crash-safe能力，即使数据库异常重启，之前提交的记录也会被记在redo log中，恢复后可以把redo log的记录再写入数据文件。

`select @innodb_flush_log_at_trx_commit`

`innodb_flush_log_at_trx_commit` 参数设置成1，表示每次事务的redo log直接持久化到磁盘，保证mysql异常重启后数据不丢失。

### binlog

binlog是逻辑架构图中Server层的日志，没有crash-safe能力。

redo log和binlog有3点不同：

1. redo log为InnoDB特有，binlog是mysql的server层实现，所有引擎都可用。
2. redo log是物理日志，记录在某个数据页上做了什么修改（硬盘分页？）；binlog是逻辑日志，记录“给id=2这行的c字段加1”。
3. redo log是循环写入；binlog是追加写入，到一定大小后切换到下一个，不会覆盖以前的。

`select @sync_binlog`

`sync_binlog` 设置成1，表示每次事务的binlog都持久化到磁盘，保证mysql异常重启后binlog不丢失。

### 执行器和InnoDB引擎在执行update语句时的内部流程

`update T set c=c+1 where ID=2;` 

1. **（执行器）** 调用引擎接口取ID=2这行
2. **（InnoDB）** 如果这一行所在数据页在内存中，直接返回，否则先从磁盘把数据页读入内存再返回。
3. **（执行器）** 拿到行数据，执行c=c+1，得到新的行，调用引擎接口写入新行。
4. **（InnoDB）** 引擎将新数据更新到内存，将更新记录到redo log，redo log处于prepare状态，告知执行器执行完成，随时可以commit事务。
5. **（执行器）** 生成操作的binlog，并写入磁盘。调用引擎的提交事务接口。
6. **（InnoDB）** 把刚刚写入的redo log改成commit状态，更新完成。

redo log的写入拆成prepare 和 commit两步，这是“两阶段提交”。

#### 两阶段提交

为了让redo log和binlog逻辑一致。

redo log大小有限，出现灾难需要恢复例如半个月前的表状态，需要binlog。

如果不使用两阶段提交，在写完第一个日志后，第二个日志没写完时crash，会导致crash恢复时，使用redo log恢复的数据（原库），与需要使用binlog恢复的数据（临时库）不一致。

例如redo log已经写完，记录了`update T set c=c+1 where ID=2`的操作，然后在写binlog没写完时crash，crash恢复后原库c=1，之后误删了表，需要用全量备份和binlog恢复，由于binlog中没有记录更新逻辑，所以恢复出来的临时库c=0，与原库不一致。

数据库扩容时也常用全量备份加binlog的方式实现，例如搭建一些备库增加系统的读能力，如果不用两阶段提交，上述的不一致就会变成出现主从数据库不一致的情况。

两阶段提交是跨系统维持数据逻辑一致时常用的一个方案。

### 一周一备vs一天一备

指全量备份。

一天一备“最长恢复时间”更短，最坏情况需要应用一天的binlog。系统对应的指标是RTO（恢复目标时间）。更频繁的全量备份需要更多存储空间。

## 03 事务隔离

事务是保证一组mysql操作要么全部成功，要么全部失败。事务支持在引擎层实现，MyISAM不支持事务，这是它被InnoDB取代的重要原因之一。

### 隔离级别

- 读未提交(read uncommitted)，一个事务还没提交，变更就可以被别的事务看到。
- 读提交(read committed)，一个事务提交之后，变更才会被其他事务看到。
- 可重复读(repeatable read)，一个事务执行过程中看到的数据，总是跟这个事务启动时看到的一致。
- 串行化(serializable)，对同一行记录，读写都会加锁，当出现读写锁冲突时，后访问事务必须等前一个事务完成才能执行。

以下表为例

```mysql
create table T(c int) engine=InnoDB;
insert into T(c) values(1);
```



|         事务A         |    事务B    |
| :-------------------: | :---------: |
| 启动事务，查询得到值1 |  启动事务   |
|                       | 查询得到值1 |
|                       |  将1改成2   |
|     查询得到值V1      |             |
|                       |  提交事务B  |
|     查询得到值V2      |             |
|       提交事务A       |             |
|     查询得到值V3      |             |

当隔离级别为读未提交时，V1=V2=V3=2。

当隔离级别为读提交时，V1=1，V2=V3=2。

当隔离级别为可重复读，V1=V2=1（事务在执行期间看到的数据一致），V3=2。

当隔离级别为串行化，B执行将1改2时会拿不到写锁，直到A提交后B才可以继续执行，然后等B提交，查询V3的事务才能继续执行，所以V1=V2=1，V3=2。

实现上，数据库里会创建一个视图，访问时以视图的逻辑结果为准。可重复读下视图在事务启动时创建，整个事务期间都用这个视图；读提交下，在每个sql语句开始时创建；读未提交直接返回记录上的最新值，没有视图概念；串行化通过加锁避免并行访问。

查看隔离级别

`show variables like 'transaction_isolation';` 

以可重复读为例，事务隔离具体实现时，每条记录在更新的时候都会同时记录一条回滚操作。记录上的最新值通过回滚可以得到前一个状态的值。

$$[1_{read-view\ A}\leftarrow2_{read-view\ B}\leftarrow3]_{回滚段}\leftarrow4_{read-view\ C}$$

当前值为4，但不同时刻启动的事务有不同的视图，同一记录在系统中存在多个版本，就是数据库的多版本并发控制（MVCC）。

当系统里没有比这条回滚日志更早的视图时，回滚日志被删除。因此，尽量不要用长事务。

长事务意味着很老的事务视图，在它提交前，数据库里它可能用到的回滚记录都必须保留，占用大量磁盘存储空间，还占用锁资源。mysql5.5及更老，回滚日志跟数据字典放在ibdata文件里，即使长事务提交，回滚段被清理，文件也不会变小。

### 事务的启动方式

1. 显式启动，`begin` 或`start transaction` ，提交`commit` ，回滚`rollback`
2. `set autocommit=0` 这条命令会关闭这条线程的自动提交。执行`select` 事务启动，持续到主动`commit` 或`rollback` 或断开连接。

建议总是`set autocommit=1` 

频繁使用事务时，使用commit work and chain语法减少交互，commit时自动启动下一个事务。

在`information_schema` 库的`innodb_trx` 表中查询长事务，以下查询持续时间超过60秒的事务。

`select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(), trx_started)) > 60;`

## 04 深入浅出索引（上）

索引用于提高查询效率。

### 常见的索引模型（数据结构）

#### 哈希表

以key-value存储数据的结构，通过哈希函数把key换算成确定的位置，把value存放在这个位置。哈希碰撞时用链表处理。

哈希表增加新的key时会很快，只需往后追加。缺点是做区间查询速度很慢。例如找出key在[x, y]区间内的所有值，就需要把区间扫描一遍。

适用于等值查询（查询key等于某个值）的场景，如Memcached以及一些NoSQL引擎。

#### 有序数组

对于等值查询和区间查询都很快，等值查询用二分法，区间查询先二分查找左边再遍历数据直到判断条件到达右边。

插入很慢，需要挪动很多数据。适用静态存储引擎，比如某年的所有人口，这类不会再修改的数据。

#### 搜索树

二叉搜索树等值查询O(log(N))，为了维持这个复杂度，需要保持平衡二叉树

大多数数据库存储不用二叉树，因为索引要写到磁盘上。

为了让查询尽量少读磁盘，就必须尽量少访问数据块，为此数据库使用N叉树，N取决于数据块大小，通常每一层数据存在一个块中。

### InnoDB的索引模型

mysql中索引在存储引擎层实现，因此并没有统一的索引标准。即使多个引擎支持同一种类型的索引，底层的数据结构可能也不同。

在InnoDB中，表根据主键的顺序以索引的形式存放，这种存储方式的表称为索引组织表。

InnoDB的索引使用B+树，数据存储在B+树中。每一个索引对应一棵B+树。

如果InnoDB引擎的表在非主键上建立索引，则至少建立两棵B+树，即主键索引和非主键索引。

主键索引的索引值是主键的值，叶子节点存的是整行数据，也称为聚簇索引（clustered index）。

非主键索引的索引值是建立索引的列的值，叶子节点存的是主键的值，在InnoDB里，也称为二级索引（secondary index）。

```mysql
create table T( id int primary key,
k int not null,
name varchar(16),
index (k))engine=InnoDB;
```

![innodb索引组织结构](https://gitee.com/huang-yd/image_repo/raw/47e91969770f7d9fc14ea7a0b3a934bcbc65d129/mysql45-lxb/04/1.png)

基于主键索引和普通索引查询的区别是：

如果`select * from T where ID=500` 只需搜索ID的B+树

如果条件`where=5` 则先搜索k索引树，得到ID，再搜索ID索引树。这个过程称为回表。

基于非主键索引的查询要多扫描一棵索引树，尽量用主键查询。

#### B+树索引的维护

如在ID索引树插入id=400的行，则需要挪动后面的数据空出位置。

如果此时R5所在数据页已满，申请新页，挪动部分数据，这个过程称为页分裂，影响性能和空间利用率（一个空间的数据分到两页中，利用率降低约50%）

相邻两页由于删除数据，空间利用率很低后，会合并数据页。

#### 哪些场景用自增主键

如果表内有普通索引，由于二级索引的叶子节点内容是主键，显然主键长度越小，叶节点越小，普通索引占用空间越小。从性能和存储空间考虑，自增主键往往更合理。

也有些场景适合用业务字段做主键，如：

1. 只有一个索引
2. 该索引必须是唯一索引

典型的key-value场景。不用考虑普通索引叶节点大小的问题。

## 05 深入浅出索引（下）

![innodb索引组织结构](https://gitee.com/huang-yd/image_repo/raw/47e91969770f7d9fc14ea7a0b3a934bcbc65d129/mysql45-lxb/04/1.png)

### 覆盖索引

如果执行`select ID from T where k between 3 and 5` 只需要查ID值，它已经在k索引树上，不需要回表。在这个查询里，索引k覆盖了查询需求，称为覆盖索引。

覆盖索引减少树的搜索次数，显著提升查询性能，是常用优化手段。

假设有一个记录市民信息的表，身份证号是唯一标识，如果需求是根据证号查询市民信息，只需要在身份证号上建立索引就足够。

如果有一个高频请求，根据身份证查姓名，则可以建立（身份证号、姓名）联合索引，它可以在这个高频请求上用到覆盖索引，不需要回表，提高性能。但是注意索引字段的维护也是有代价的。

### 联合索引的字段顺序

只要满足最左前缀，就可以利用索引加速检索。最左前缀可以是联合索引的最左N个字段，也可以是字符串索引的最左M个字符。

索引支持最左前缀，因此模糊查询时，尽量明确条件的开头。也因此当有了（a，b）联合索引后，一般不需要单独在a上建立索引。

因此，**第一原则是，如果通过调整字段顺序，可以少维护一个索引，那么这个顺序往往是优先考虑采用的。** 

以市民信息为例，为高频请求创建（身份证号，姓名）这个联合索引后，可以用它支持“根据身份证号查地址”。

如果有(a,b)的联合索引，又有条件里只有b的高频语句和只有a的高频语句，就需要同时维护(a,b)、(b)两个索引。这时**考虑的原则是空间** 。例如a字段比b字段大，就建立(a,b)和(b)，反之建立(b,a)和(a)

### 索引下推（index condition pushdown）

有联合索引(name, age)，有sql语句

`select * from tuser where name like '张%' and age=10 and ismale=1;`

这个语句在搜索索引树时只能用“张”（最左匹配，顺序是name，age，在name这里就模糊了，后面age=10用不上），之后：

mysql5.6前，不看age，直接回表，对比字段；

mysql5.6引入索引下推优化，在索引遍历过程中，对索引包含的字段先做判断，即看age，过滤掉不满足条件的记录，减少回表次数。

## 06 全局锁和表锁

mysql的锁大致分为全局锁、表级锁、行锁。

### 全局锁

对整个数据库实例加锁。mysql加全局读锁的命令是`Flush tables with read lock` （FTWRL），之后其他线程的数据更新语句（增删改）、数据定义语句（建表、改表结构）和更新类事务提交语句会被阻塞。

**典型使用场景是全库逻辑备份。**

全局读锁让整个库只读，可能导致业务停摆、主从延迟。

官方逻辑备份工具是mysqldump，当mysqldump使用参数-single-transaction时，导出数据前会启动一个事务保证拿到一致性视图，由于支持MVCC，这个过程中数据可以正常更新。

MyISAM不支持事务，更不支持可重复读的隔离级别，因此single-transaction方法只适用所有表都使用支持事务引擎的库，否则只能通过FTWRL方法备份。

不建议使用`set global readonly=true` 来设置全库只读。readonly的值可能用做其他逻辑，修改global变量影响面更大；在异常处理上，如果执行FTWRL后客户端异常断开，mysql会自动释放全局锁，而设置readonly之后，客户端断开mysql会保持readonly状态。

### 表级锁

有两种，表锁和元数据锁（meta data lock，MDL）

表锁语法为`lock tables t1 read, t2 write` ，可以用`unlock tables` 主动释放，客户端断开也会自动释放。执行后其他线程写t1，读写t2会阻塞，本线程在`unlock tables` 前也只能读t1，读写t2。

InnoDB支持行锁，一般不用lock tables。

一般只有引擎不支持行锁才会用到表锁。

MDL在访问一个表时会自动加上，在mysql5.5中引入。当对一个表增删改查时加MDL读锁；当对表做结构变更时加MDL写锁。

MDL是server层的锁。读锁之间不互斥，多个线程可以同时对一张表增删改查（这里理解是如果同时对一段记录读和写，更具体的交给引擎处理）；读写锁互斥，写锁之间互斥，保证变更表结构操作的安全性。

| Session A | Session B | session C | session D |
| :----: | :----: | :----: | :----: |
| begin; |      |      |      |
| Select * from t limit 1; |      |      |      |
|      | Select * from t limit 1; |      |      |
|      |      | alter  table t add f int;(blocked) |      |
|      |      |      | select * from t limit 1;(blocked) |

session A先启动，对表t加MDL读锁，session B也加读锁，不互斥，正常运行。

session C要加MDL写锁，因为A的MDL读锁还没释放，所以session C被阻塞。申请写锁的请求被加入优先级队列。

session D要申请MDL读锁，也会被加入优先级队列，因为C的写锁优先级更高，所以D只能排队，一直被阻塞，此时这个表已经完全不可以读写了。

如果表上查询频繁，且客户端有重试机制，超时后会另开新的session，那么这个库的线程很快就爆满。

事务中的MDL锁，在语句执行开始时申请，等事务提交后再释放，因此要避免长事务。

### 如何安全地给小表加字段

1. 解决长事务。在mysql的`information_schema` 库的`innodb_trx` 表中查询执行中的事务，如果要做DDL（数据定义语句，改表结构，DML数据操作语句，增删改查）的表刚好有长事务在执行，需要推迟DDL，或kill掉长事务。
2. `alter table` 里设定等待时间，在等待时间里拿不到就放弃，之后开发人员再重试命令。MariaDB合并了AliSQL的这个功能，目前这两个开源分支都支持DDL NOWAIT/WAIT n的语法。

```mysql
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tlb_name WAIT N add column ...
```

## 07 行锁功过：怎么减少行锁对性能的影响？

### 两阶段协议

行锁在引擎层由各个引擎自己实现。对不支持行锁的引擎比如MyISAM，并发控制只能用表锁，同一张表上任何时刻都只能有一个更新在执行。

在InnoDB事务中，行锁在需要时加上，事务结束时释放。这就是两阶段锁协议。

因此，如果事务中需要锁多个行，调整语句顺序，把最可能造成锁冲突、最有可能影响并发度的锁的申请时机往后放，减少锁那一行的时间。

### 死锁和死锁检测

|                 事务A                  |               事务B                |
| :------------------------------------: | :--------------------------------: |
| `begin;update t set k=k+1 where id=1;` |              `begin;`              |
|                                        |  `update t set k=k+1 where id=2;`  |
|   `update t set k=k+1 where id =2;`    |                                    |
|                                        | `update t set k=k+1 where id = 1;` |

A、B互相等待对方的行锁释放，出现死锁。

出现死锁后两种策略：

1. 等待直到超时，超时时间通过`innodb_lock_wait_timeout` 参数设置，InnoDB中默认值50s。
2. 死锁检测，发现死锁后主动回滚死锁链条中某一个事务，让其他事务继续执行。设置参数`innodb_deadlock_detect` 为 `on` 开启。

第一种策略设置太长在线服务无法接受，太短容易误把正常等待的事务误伤。主要用第二种策略。

死锁检测需要消耗cpu，单条线程检测的时间复杂度是O(n)，n为要更新同一行的线程数，因此总的时间复杂度为$$O(n^2)$$

解决方案一是如果确保这个业务一定不会死锁，临时关闭死锁检测。如果判断错误，可能出现大量超时，有损业务；二是控制并发度，一般在中间件、服务端、修改mysql源码里实现，基本思路是对相同行的更新，进入引擎前排队，InnoDB内部就可以避免大量的死锁检测工作。

也可以从逻辑上优化，比如一条记录分成十条记录的和，更新同一记录的冲突概率变为1/10，减少锁等待个数以及死锁检测的cpu消耗，业务上需要详细设计和特殊处理。

## 08 事务是隔离的还是不隔离的？

在InnoDB中，`begin/start transaction` 不是事务的起点，在执行到第一个操作InnoDB表的语句，事务才真正启动。在可重复读下，`begin\start transaction` 在第一个快照读的时候，得到一致性视图。想要马上启动一个事务，使用`start transaction with consistent snapshot` 命令，得到一致性视图。

mysql里有两个“视图”

1. view，用查询语句定义的虚拟表，查询方法和表一样，在调用时执行查询语句并生成结果。
2. InnoDB在实现MVCC时用的一致性视图，consistent read view，用于支持读提交和可重复读。

### 可重复读下的快照

在可重复读级别下，事务启动时拍下一个基于整个库的快照。

InnoDB每个事务有唯一的事务id，transaction id，在事务开始时向InnoDB事务系统申请，按申请顺序严格递增。

数据表中的每行记录，可能有多个版本，每次事务更新数据，都会生成一个新的数据版本，并且数据版本的事务id，即`row trx_id` 被赋值为transaction id

![行状态变更图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/08/1.png)

图中3个虚线箭头就是前文提到的undo log，V1、V2、V3在物理上不存在，需要V2的时候通过V4和undo log依次执行U3、U2计算。

InnoDB利用“所有数据都有多个版本”的特性，实现了秒级创建快照的能力。在实现时，为每个事务构造一个视图数组，保存事务启动瞬间，所有启动但未提交的事务id。数组最小值记为低水位，当前系统里已经创建过的事务id的最大值加1记为高水位。当前事务的id也会加入数组内。

这个视图数组和高水位，组成了当前事务的一致性视图。

数据版本的可见性规则是基于数据的row trx_id和这个视图对比结果得到。在当前事务启动的瞬间

1. 一个数据版本的row trx_id，如果小于低水位，则这个版本是已提交的事务生成的，对当前事务可见；
2. 如果大于等于高水位，则是由将来启动的事务生成，不可见；
3. 如果低水位<=row trx_id<高水位，则看是否在视图数组中，在的话不可见，反之可见。除此之外，当前事务改动生成的数据版本必然对自己可见。

~~简单的东西复杂化~~ 换句话说成下面的规则更直观：对一个事务视图，看一个数据版本，除了自己的更新总是可见外有3种情况

1. 版本未提交，不可见（将来启动的事务或者在视图数组里的事务修改的）
2. 版本已提交，但是是在视图创建后提交，不可见（将来启动的事务或者在视图数组里的事务提交的）
3. 版本已提交，而且是在视图创建前提交，可见（~~废话一句~~ 可能row trx_id大于低水位，也可能小于低水位，但一定不在视图数组内。）

### 更新逻辑

**更新数据都是先读后写，读只能读当前的值，称为“当前读”** 。

用以下InnoDB下的流程举例一致性视图、当前读、行锁的逻辑

|                     事务A                     |                            事务B                            |                            事务C                             |
| :-------------------------------------------: | :---------------------------------------------------------: | :----------------------------------------------------------: |
| `start transaction with consistent snapshot;` |                                                             |                                                              |
|                                               |        `start transaction with consistent snapshot;`        |                                                              |
|                                               |                                                             | `start transaction with consistent snapshot;update t set k=k+1 where id=1;` |
|                                               | `update t set k=k+1 where id=1;select k from t where id=1;` |                                                              |
|     `select k from t where id=1;commit;`      |                                                             |                          `commit;`                           |
|                                               |                          `commit;`                          |                                                              |

事务C先获得这条记录的写锁，并且在事务B更新的时候仍未释放，因此B被阻塞，在C释放锁后B加锁，进行当前读，然后更新；A没有更新语句，不做当前读，按照一致性视图的规则，B和C都属于高水位之后的事务，对A不可见。

### 可重复读和读提交的实现

可重复读的核心是一致性读；当事务更新的时候，只能用当前读；如果当前记录的行锁被占用，需要进入锁等待。

读提交和可重复读的主要区别:

1. 可重复读在事务第一次快照读（原文是事务开始）时创建一致性视图，之后事务里的其他查询共用这个视图。`begin\start transaction` 并没有真正开始事务，执行第一条操作表的语句才开始事务，开始第一次快照读才创建一致性视图，如果事务第一条语句是delete\update，是不会创建一致性视图的，直到select才创建一致性视图，在select之前，别的事务insert数据，之后用select是可以看到的。
2. 读提交在每一个语句执行前重新算出新的视图。

`start transaction with consistent snapshot` 意思是从这句开始创建持续整个事务的一致性视图，在读提交级别下，没有意义，等价于`start transaction`

表结构不支持可重复读，因为没有对应行数据，也没有row trx_id，只能遵循当前读的逻辑。

Mysql8.0把表结构放在InnoDB字典里，以后可能支持表结构可重复读。

## 09 普通索引和唯一索引，应该怎么选择？

唯一索引指用于创建索引的列的值是唯一的。

### 查询过程

对查询来说，普通索引在查找到满足条件的记录后，继续遍历查找下一个记录，直到条件不满足。

对唯一索引来说，因为有唯一性，所以找到满足条件的记录直接停止检索。

两者的性能差距微乎其微。在InnoDB中，数据页的默认大小是16KB，InnoDB读写数据到内存是以页为单位的，因此对普通索引，找到记录时，所需的数据通常都已经在内存里，一次“查找和判断下一条记录”的操作，通常不需要IO，只需要一次指针寻找和一次计算。如果满足条件的记录正好是一页的最后一条，则可能需要IO，但是对于整型字段，一页可以存放近千个key，需要IO的概率很低。

### 更新过程

当需要更新数据页时，如果已经在内存，就直接更新；如果不在，为了避免IO操作，在不影响数据一致性的前提下，InnoDB将更新操作缓存在change buffer，下一次查询需要访问此数据页再执行change buffer中相关的操作，得到最新的结果，这个过程称为merge。

change buffer可以持久化到磁盘。

merge的时机为访问数据页，系统后台线程定期，数据库正常关闭时。

change buffer减少磁盘IO可以明显提升性能，并且减少数据读入内存占用buffer pool，避免占用内存，提高内存利用率。

对唯一索引，要判断唯一性，因此数据页必须在内存中，用不上change buffer。只有普通索引可以用change buffer。

change buffer的大小通过参数`innodb_change_buffer_max_size` 设置，它为50的时候，表示change buffer的大小最多占用buffer pool的50%。

如果更新的目标页在内存中，普通索引和唯一索引更新的消耗几乎没有区别，唯一索引只多一个判断的cpu时间。

如果更新目标页不在内存，则唯一索引必须读数据页，有磁盘IO，是数据库成本最高的操作之一，普通索引只需更新change buffer。

### change buffer使用场景

对写多读少的业务，页面写完后马上被访问的概率小，change buffer可以缓存较多操作，每次IO的收益较大。常见账单、日志系统。

对更新后很快查询的业务，操作记录在change buffer后马上触发merge，不会减少磁盘IO，还会增加维护change buffer的代价，反而降低性能。

### 索引选择小结

查询过程没有区别，更新过程普通索引更优，尽量选普通索引，对于更新完就查的业务，关闭change buffer。注意先保证业务正确性，如果业务代码保证不会写入重复数据，再讨论性能，如果业务不能保证，或本身就要求数据库做唯一性约束，还是要用唯一索引。

### Change buffer和redo log

`insert into t(id,k) values(id1,k1),(id2,k2);`

执行这条语句，假设k索引树找到位置后，k1所在数据页在内存中，k2所在数据页不在。

<img src="https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/09/1.png" alt="带changebuffer的更新过程" style="zoom: 200%;" />

1. page1在内存中，直接更新
2. page2不在内存，在change buffer中记录对page2的操作
3. 前两步写入redo log

做完以上事务就完成了。

ibdata1和t.ibd是磁盘数据，虚线表示在适当的时候写入。

如果读发生的时候，内存的数据都还在，那么

1. 读page1直接从内存返回，跟redo log无关。
2. 读page2，把page2从磁盘读入内存，应用change buffer里的操作，生成正确的数据版本并返回。

>redo log主要节省的是随机写磁盘的IO消耗（转成顺序写），change buffer主要节省的是随机读磁盘的IO消耗。

这里理解是不需要频繁且随机地把操作记录到磁盘中，读redo log所在的数据页时一次性顺序写入操作，节省随机写；change buffer避免随机的更新操作频繁地读入并修改数据页，节省随机读。

### merge执行流程

1. 磁盘读数据页到内存
2. 从change buffer 里找到这个数据页的记录，依次应用，得到新的数据页
3. 写redo log，包含数据的变更和chang buffer的变更。

到此merge结束。磁盘上的数据页和change buffer还没有更新，后续写回磁盘属于另外的过程。

## 10 mysql为什么会选错索引

### 纠错

在mysql8.0下实验，确定隔离级别为可重复读，引擎为InnoDB。

```mysql
CREATE TABLE `t` (
`id` int(11) NOT NULL, `a` int(11) DEFAULT NULL, `b` int(11) DEFAULT NULL, PRIMARY KEY (`id`),
KEY `a` (`a`),
KEY `b` (`b`)
) ENGINE=InnoDB;

delimiter ;;
create procedure idata() begin
  declare i int;
  set i=1;
  while(i<=100000)do
insert into t values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
```



|                   Session A                   |                         session B                          |
| :-------------------------------------------: | :--------------------------------------------------------: |
| `start transaction with consistent snapshot;` |                                                            |
|                                               |               `delete from t;call idata();`                |
|                                               |                                                            |
|                                               | `explain select * from t where a between 10000 and 20000;` |
|                   `commit;`                   |                                                            |

课件说session B的查询不会再选择索引a，但实际上仍然会选择，性能并不受影响。

`select * from t where a between 10000 and 20000; /*Q1*/`

`select * from t force index(a) where a between 10000 and 20000;/*Q2*/`

课件说Q1走全表扫描，Q2用了索引a，Q2比Q1快一倍，但实际上两者执行是一样的，explain也是一样的。

可能是mysql8.0.25做了优化。

### 优化器的逻辑（以下内容应该是针对mysql8.0.25以下版本，因为书中例子我复现不成立）

优化器选择索引，找到最优执行方案，用最小代价执行语句。扫描行数是影响代价因素之一，扫描越少，访问磁盘越少，消耗cpu也越少。还会结合是否使用临时表、是否排序、是否需要回表等因素。

一个索引上不同值的个数，称为基数，基数越大，索引区分度越好。用`show index from tbl_name;` 可以查看索引基数。

InnoDB默认选择N个数据页，统计页面上的不同值，得到一个平均值，乘以索引的页面数，得到索引基数，称为采样统计。当数据表变更行数超过1/M的时候会自动触发重新做一次采样统计。

mysql中有两种存储索引统计的方式，通过参数`innodb_stats_persistent` 选择：

- 设置为1（on）的时候，表示统计信息持久化，默认N=20，M=10，默认选择on
- 设置为0（off）的时候，统计信息只存在内存，默认N=8，M=16

在mysql错误判断扫描行数（explain查看）的时候，可以使用`analyze table tbl_name;` 命令修正统计信息。

### 以下内容在mysql8.0.25也成立

除了扫描行数，排序也会影响索引的选择。

`select * from t where (a between 1 and 1000) and (b between 50000 and 100000) order by b limit 1;` 

选择索引a，需要扫描索引a的前1000个值，然后回表，取值，判断；选择b，需要扫描50001行。理应选择索引a。

explain实验结果：

```mysql
mysql> explain select * from t where (a between 1 and 1000) and (b between 50000 and 100000) order by b limit 1;
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+------------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows  | filtered | Extra                              |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+------------------------------------+
|  1 | SIMPLE      | t     | NULL       | range | a,b           | b    | 5       | NULL | 49111 |     1.02 | Using index condition; Using where |
+----+-------------+-------+------------+-------+---------------+------+---------+------+-------+----------+------------------------------------+
1 row in set, 1 warning (0.00 sec)

```

索引选择错误，判断要扫描49111行。

查询时使用`select * from t force index(a)...` 确实会更快，扫描行数也准确，但是性能相差不大，在mac上只相差0.07秒。

优化器选择索引b时因为使用索引b可以避免排序（索引本身有序），所以即使扫描行数多，优化器也判断代价更小。

当改为`...order by b,a limit 1;` 后两个索引都需要排序，扫描行数成为影响的主要条件，此时优化器选择索引a。

所以，优化器选择索引错误时有三种方法：

1. 强制选择索引，`force index`
2. 从数据特征上，在语义不变的情况下诱导优化器，比如`order by b` 变为`order by b,a`
3. 新建更合适的索引，或在分析后发现错误索引多余时，直接删掉错误的索引。

## 11 怎么给字符串字段加索引

### 给字符串创建前缀索引应该用多长的前缀

mysql支持前缀索引，如`alter table SUser add index index2(email(6));` 只取email字段的前6个字符做索引，默认是取整个字符串。

建立索引时关注区分度，区分度越高，基数越大，重复的键越少，索引效果越好。

可以先算出列上有多少个不同的值

`select count(distinct email) as L from SUser;`

然后取不同前缀看有多少不同的值

```mysql
mysql> select
count(distinct left(email,4))as L4, count(distinct left(email,5))as L5, count(distinct left(email,6))as L6, count(distinct left(email,7))as L7,
from SUser;
```

设置一个可接受的损失比例，如5%，从上述结果中找出不小于95%L的值，然后选需要更少字符的。

### 前缀索引的缺点

用不上覆盖索引对性能的优化。如id为主键时

`select id,email from SUser where email='...';` 可以用上覆盖索引的优化，不需要回表。而如果用前缀索引，由于mysql不确定前缀索引的定义是否包含了完整字段的信息，不得不回表取整行再判断email字段的值。

### 需求只有等值查询时的优化

#### 倒序存储

例如身份证号前6位很多相同，倒过来存储就可以使用前缀索引来节省空间以及提高查询效率

`select field_list from t where id_card=reverse('input_id_card_string');` 

实践中在倒序存储以及建立前缀索引前，记得用`count(distinct)` 的方法验证区分度。

#### 加一个冗余的hash字段

`alter table t add id_card_crc int unsigned, add index(id_card_crc);`

每次插入新的记录，同时用crc32()函数得到校验码。由于crc32可能冲突，判断时还要判断id_card是否精确相同。

id_card_crc需要索引的长度只有4个字节，比身份证的长度小很多。

#### 倒序和hash异同点

都不支持范围查询，都只能等值查询。

不同点主要有三方面：

1. hash需要增加一个字段，倒序不会消耗额外存储空间，但是如果倒序存储建立前缀索引需要的长度不够短，那么前缀长度的消耗和额外建立hash字段的消耗可能也会抵消。
2. 倒序每次读写要调用reverse函数，hash需要调用crc32函数，reverse函数消耗的cpu更小些。
3. hash的查询性能更好更稳定。crc32冲突概率小，可以认为每次查询平均扫描行数接近1，倒序存储用的还是前缀索引，相比下可能还是会增加扫描行数。

## 12 为什么mysql会“抖”一下

"抖"，指一条sql语句，正常执行特别快，但有时特别慢，很难复现，持续时间很短。

当内存数据页与磁盘数据页内容不一致，称这个内存页为脏页，一致称为干净页。

“抖”可能是平时在写内存和redo log，抖的时候在刷脏页（flush），把内存的内容同步到磁盘。

flush的四种场景：

1. redo log写满，所有更新都被堵塞，checkpoint需要前推，移动位置之间的日志对应的脏页需要写到磁盘。
2. 内存不够，需要淘汰数据页，如果淘汰脏页，就需要先写入磁盘。
3. mysql空闲的时候。
4. mysql正常关闭之前会把所有脏页flush到磁盘。

InnoDB用buffer pool管理内存，缓冲池中的内存页有仍未使用、干净页、脏页三种状态。

### InnoDB刷脏页的控制策略

刷脏页是常态，但两种情况会明显影响性能：

1. 一个查询要淘汰的脏页太多，导致响应时间明显变长。
2. 日志写满，所有更新堵住，写性能跌到0。

InnoDB需要知道所在主机的IO能力，控制刷脏页的速度，`innodb_io_capacity` 参数告诉InnoDB的磁盘能力，建议设置成磁盘的IOPS，这个值可以通过fio工具测试。

同时InnoDB不能占用全部磁盘IO能力，磁盘还要响应用户请求。

InnoDB刷脏页的速度主要参考脏页比例和redo log写盘速度。

`innodb_max_dirty_pages_pct` 参数表示脏页比例上限，默认75%，InnoDB会根据当前的脏页比例M，用F1(M)算出一个[0,100]的数字。

InnoDB每次写入redo log都有一个序号，当前写入序号跟checkpoint对应序号之间的差值为N，根据N算出[0,100]的数字，算法为F2(N)，N越大结果越大。

$$R=\max{(F1(M), F2(N))}$$ ，刷脏页的速度为$$R\%\ *\ innodb\_io\_capacity$$

![InnoDB刷脏页速度策略](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/12/1.png)

要避免mysql“抖”，要合理设置`innodb_io_capacity` ，并关注脏页比例，不要让它经常接近75%

脏页比例通过`Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total` 得到

```mysql
mysql> select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
select @a/@b;
```

### mysql刷脏页的“空间关联”机制

一旦一个查询在执行过程中需要先flush一个脏页，如果这个脏页旁边的数据页也是脏页，也会一起flush，并且还会看邻居的邻居，一直外推。

在使用机械硬盘时，这个策略很有意义，减少很多随机IO，减少磁盘物理上寻址移动的时间。机械硬盘的IOPS一般只有几百，减少随机IO可以大幅提升性能。

在使用SSD这类IOPS比较高的设备时，意义不大。

这个策略用`innodb_flush_neighbors` 参数控制，设为1表示会flush邻居脏页。

mysql8.0中，上述参数默认为0。

## 13 为什么表数据删掉一半，表文件大小不变（数据库表的空间回收）

针对InnoDB引擎讨论。一个InnoDB表包含两部分：表结构定义和数据。在mysql8.0以前，表结构存在.fm后缀文件里，在8.0后，允许把表结构定义放在系统数据表中。

### innodb_file_per_table

表数据既可以存在 共享表空间 里，也可以是单独文件。

`innodb_file_per_table `设置为off，表示表的数据放在系统共享表空间，跟数据字典放一起；设置为on，表示每个InnoDB表数据存在一个.ibd后缀文件里。

在mysql5.6.6开始，默认为on

建议一直设置为on，在不需要这个表的时候，通过`drop table` 系统会直接删除ibd文件，而如果放在 共享表空间 中，即使表删掉空间也不会回收。接下来基于这个参数设置为on展开讨论。

### 数据删除流程

InnoDB的数据用B+树存储，要删除一个记录，InnoDB引擎只会把这个记录标记为删除，位置可复用，如果之后再插入一条符合范围条件的记录，可能会复用这个位置，但是磁盘文件不会缩小。

如果InnoDB整个数据页上的所有记录被删除，那么整个数据页就可以被复用。

如果相邻两个数据页利用率都很小，会合并数据页，并标记另一个可复用。

如果用`delete` 把整个表删除，所有数据页都会被标记可复用，但是磁盘文件不会变小。

`delete` 不能回收表空间，只会标记“可复用”，这些可以复用但是没有被使用的空间，看起来就像“空洞”。

### 插入数据也会造成“空洞”

如果数据按照索引顺序递增插入，那么索引是紧凑的。

如果随机插入，可能造成数据页分裂。

假设pageA已满

![插入数据导致页分裂](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/13/1.png)

可见pageA的位置上留下了空洞（可能不止一个）

另外，更新索引上的值，也可以理解为删除旧的值，插入新值，也会造成空洞。

### 重建表

重建表可以去掉空洞，收缩空间。

实际上，InnoDB重建表，不会把整张表占满，每个页留了1/16给后续的更新使用，也就是说重建完的表在空间上并不是最紧凑的。

#### mysql5.5之前

表A需要去掉表中的空洞。我们可以新建与A结构相同的B，然后按照主键ID递增顺序，把数据一行一行从A读出再插入B。把B作为临时表，A导入B完成后，用B替换A，就完成了收缩空间的目的。

`alter table A engine=InnoDB` 执行流程跟上述差不多，mysql自己完成转存数据、交换表名、删除旧表。这是DDL（数据定义语句）操作。

临时表由server层创建。

这个过程如果有新数据写入到A，会造成丢失，因此整个DDL过程中，表A不能更新，即这个DDL不是online的。

![锁表DDL](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/13/2.png)

#### mysql5.6之后

引入online DDL。

>1. 建立一个临时文件，扫描表A主键的所有数据页;
>2. 用数据页中表A的记录生成B+树，存储到临时文件中;
>3. 生成临时文件的过程中，将所有对A的操作记录在一个日志文件(rowlog)中，对应的是图 中state2的状态;
>4. 临时文件生成后，将日志文件中的操作应用到临时文件，得到一个逻辑数据上与表A相同的 数据文件，对应的就是图中state3的状态;
>
>5. 用临时文件替换表A的数据文件。

![online DDL](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/13/3.png)

由于rowlog和重放操作的存在，重建过程中允许对A增删改。

临时文件是在InnoDB内部创建。

不管是否online，都很消耗cpu和io，online操作可以考虑在业务低峰使用。

##### MDL写锁退化

在上述online DDL流程中，alter语句启动时要获取MDL写锁，但是在真正拷贝数据时退化成MDL读锁，不阻塞增删改操作，同时要禁止其他线程对这个表做DDL，所以不能直接解锁。

online DDL最耗时的是拷贝数据的操作，这个过程上的是MDL读锁，可以接受增删改，因此相对整个DDL过程，写锁锁上的时间非常短，对业务来说可以认为是online的。

#### online和inplace

mysql5.5之前，存放A临时数据的位置叫tmp_table，是server层创建的临时表。

mysql5.6之后，根据A重建的数据放在tmp_file，是InnoDB内部创建的临时文件，整个过程在InnoDB内部完成，对server层来说没有把数据挪动到临时表，因此是一个inplace操作。

这里感觉没说清楚，临时表难道只存在于内存，不会有磁盘文件？不然的话临时表不也是一个文件吗？

mysql5.6后，重建表`alter table t engine=InnoDB` 其实是`alter table t engine=innodb,ALGORITHM=inplace;` ,对应的是mysql5.5之前的拷贝表方式`alter table t engine=innodb,ALGORITHM=copy;` 

加全文索引的操作是inplace的，会阻塞增删改操作，是非online的。

1. DDL如果是online的，一定是inplace的
2. inplace的不一定是online的，截止mysql8.0，添加全文索引fulltext index和空间索引spatial index就属于这种情况。

#### optimize table、analyze table、alter table区别

- mysql5.6起，默认就是上述重建（recreate）表的流程
- analyze table只是对索引信息重新统计，没有修改数据，这个过程加MDL读锁。
- optimize table等于recreate+analyze

## 14 count(*)这么慢怎么办？

### count(*)的实现方式

不同引擎实现方式不一样

- MyISAM把一个表的总行数存在磁盘上，执行count(*)时直接返回
- InnoDB把数据逐行从引擎里读出计数

这篇讨论没有过滤条件的count，如果加了where，myisam也不能返回这么快。

InnoDB的事务默认隔离级别是可重复读，代码上通过MVCC实现。每一行都要判断是否对这个会话可见，因此只能逐行读出计算count。

**在保证逻辑正确前提下，尽量减少扫描数据量，是数据库系统设计的通用法则。** InnoDB是索引组织表，主键索引的叶节点是数据，普通索引叶节点是主键值，比主键索引小很多，所以mysql优化器会找到最小的索引树遍历得到count。

`show table status` 输出的TABLE_ROWS是从索引统计值得到的，索引统计值是采样估算的，误差可能达到40%到50%。

### InnoDB表我们只能自己计总行数

基本思路是找一个地方把行数存起来。

#### 缓存系统保存计数

用redis保存表的总行数，读写都很快，问题是

- redis异常重启时，缓存计数的最新操作可能会丢失。但是异常重启不多见，重启后做count(*)再更新计数，成本仍可以接受
- 由于时序问题，即使redis正常工作，计数值也可能不准。比如在插入新行和写redis中间，读取最新行数以及最新更新的几行，得到的行数跟最新更新的行是对应不上的。

#### 用数据库保存计数

InnoDB支持redo log，崩溃恢复不丢数据。

用事务来解决时序问题。

| 时刻 |             会话A              |                            会话B                             |
| :--: | :----------------------------: | :----------------------------------------------------------: |
|  T1  |                                |                                                              |
|  T2  | begin；<br />表C中计数值加1；  |                                                              |
|  T3  |                                | begin；<br /> 读表C计数值；<br /> 查询最近100条记录；<br />  commit； |
|  T4  | 插入一行数据R；<br /> commit； |                                                              |

可见如果是redis，因为没有事务，会出错，而InnoDB用事务可以得到正确结果。

### 不同count的性能

以下基于InnoDB

count是一个聚合函数，对返回的结果集，逐行判断，如果count的参数不是NULL，就累计值加1，否则不加，最后返回累计值。

`count(*)、count(主键id)、count(1)` 都表示返回满足条件的结果集总行数，`count(字段)` 表示返回满足条件的数据行里，参数“字段”不是NULL的总个数。

分析性能的原则：

1. server层要什么就给什么
2. InnoDB只给必要的值
3. 优化器只优化了`count(*)` 的语义为取行数，其他优化并没有做

- `count(主键id)` ，引擎遍历整张表，取出每一行的id值，返回给server层，server层判断id不可能为NULL，直接按行累加值
- `count(1)` 遍历整张表，不取值，返回给server层，server层对每一行，放一个数字“1”进去，判断是不可能为空的，按行累加。比`count(主键id)` 快，后者涉及解析数据行和拷贝字段值
- `count(字段)` 如果字段定义是not null，则遍历整张表，取出字段值，返回给server层，server层判断不可能为null，按行累加；如果字段定于允许为null，则server层判断有可能是null，需要把字段值取出来判断不是null才累加
- `count(*)` 专门做了优化，不取值，server层判断肯定不是null，按行累加

效率排序`count(字段)` < `count(主键id)` < `count(1)`  ≈ `count(*)` 

## 15 答疑（一）：日志和索引相关

### 两阶段提交的不同瞬间crash怎么办

`update T set c=c+1 where id=2;`

两阶段提交参考02

![两阶段提交示意图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/15/1.png)

> 1. **（执行器）** 调用引擎接口取ID=2这行
> 2. **（InnoDB）** 如果这一行所在数据页在内存中，直接返回，否则先从磁盘把数据页读入内存再返回。
> 3. **（执行器）** 拿到行数据，执行c=c+1，得到新的行，调用引擎接口写入新行。
> 4. **（InnoDB）** 引擎将新数据更新到内存，将更新记录到redo log，redo log处于prepare状态，告知执行器执行完成，随时可以commit事务。
> 5. **（执行器）** 生成操作的binlog，并写入磁盘。调用引擎的提交事务接口。
> 6. **（InnoDB）** 把刚刚写入的redo log改成commit状态，更新完成。
>
> redo log的写入拆成prepare 和 commit两步，这是“两阶段提交”。

commit语句执行的时候，包含commit步骤。

时刻A，mysql崩溃，redo log未提交，binlog还没写，崩溃恢复时，这个事务回滚，binlog没写，所以也不会传到备库。

崩溃恢复时的判断规则：

1. 如果redo log里事务已经有了commit标识，直接提交。
2. 如果redo log里面事务只有完整的prepare，判断对应的事务binlog是否完整，是则提交事务，否则会滚事务。

时刻B对应2中binlog完整，所以崩溃恢复后事务会提交。

### mysql怎么知道binlog完整

一个事务的binlog有完整的格式

- statement格式的binlog最后有COMMIT
- row格式的binlog最后会有一个XID event

在mysql5.6.2后引入binlog-checksum参数，mysql通过checksum校验的结果确认事务binlog完整性

### redo log和binlog怎么关联

有一个共同的数据字段叫XID。崩溃恢复时顺序扫描redo log：

- 碰到有commit的redo log，直接提交事务
- 碰到只有prepare没有commit的redo log，拿着XID去binlog找对应的事务

这里没说如果binlog没有对应事务怎么办，按照前文应该是回滚。

### 处于prepare的redo log加完整binlog，重启就能恢复，mysql为什么要这么设计？

这里问题感觉没说清楚，问的应该是为什么prepare的redo log加完整binlog不回滚而是提交？

看前文时刻B，这时候mysql崩溃，binlog已经写入，之后会被从库或用这个binlog恢复出来的库使用，因此为了保证数据一致性，主库上也要提交这个事务。

### 为什么要两阶段提交？先写完redo log再写binlog，崩溃恢复的时候必须两个日志都完整才恢复，是否可行？

两阶段提交是经典的分布式系统问题，非mysql独有。

对InnoDB，如果redo log已经提交，则事务不能回滚（如果这时还允许回滚可能会覆盖掉别的事务的更新）。如果redo log不prepare，直接提交，这时binlog写入失败，因为不能回滚，数据和binlog不一致。

### 不引入redo log，只用binlog做崩溃恢复和归档行不行？

历史原因，mysql原生引擎MyISAM设计时就不支持崩溃恢复。

InnoDB作为mysql插件加入之前，已经是一个支持崩溃恢复和事务的引擎了。接入mysql后发现binlog不支持崩溃恢复自然就用自有的redo log。

实现上的原因，binlog是逻辑日志，没有能力恢复数据页。InnoDB使用WAL技术，执行事务时，写完内存和日志，事务就算完成。之后崩溃要依赖于日志恢复数据页，binlog做不到。

如果要优化binlog让它记录数据页的更改，那跟做一个redo log没区别。

### 能不能只用redo log

redo log大小有限，循环写，不能归档。比如要恢复到半个月之前，只能依赖binlog。

除此之外，mysql本身以及很多公司业务都依赖于binlog。

### redo log一般多大

太小的话很快写满，会经常强行刷redo log，WAL机制的能力发挥不出来，并引起change buffer merge，表现就是数据库写性能经常下跌。

常见的几个T的硬盘直接将redo log设置成4个1G的文件。

### 数据最终写入磁盘，是由redo log写吗？

redo log并没有记录数据页的完整数据，并没有能力去更新磁盘数据页。

1. 如果是正常运行的实例，脏页被写入磁盘，这个过程跟redo log毫无关系。

2. 在崩溃恢复的场景中，InnoDB如果判断一个数据页在崩溃时丢失了更新，会把数据页读入内存，然后让redo log更新内存内容，数据页变为脏页，回到1。

### redo log buffer是什么？

事务要在commit之后才写到redo log文件里。而一个事务的执行过程中，可能有多个语句，写多次日志，这时就把日志先写到内存，即redo log buffer。

真正把日志写到redo log文件即ib_logfile+数字文件，是在执行commit语句时。

以上说的是事务执行过程中不会主动去写磁盘，减少不必要的IO，但是如果内存不够、其他事务提交等情况，可能会被动写入磁盘。

## 16 order by怎么工作

### 全字段排序

`explain` 语句中Extra内容有“Using filesort”表示需要排序

mysql会给每个线程分配一块用于排序的内存，称为sort_buffer

```mysql
CREATE TABLE `t` (
`id` int(11) NOT NULL,
`city` varchar(16) NOT NULL,
`name` varchar(16) NOT NULL,
`age` int(11) NOT NULL,
`addr` varchar(128) DEFAULT NULL, 
PRIMARY KEY (`id`),
KEY `city` (`city`)
) ENGINE=InnoDB;
```



`select city,name,age from t where city='杭州' order by name limit 1000;`

通常情况下，这个语句执行流程如下：

1. 初始化sort_buffer，确定放入name，city，age三个字段
2. 从索引city找到第一个满足city条件的主键id
3. 到主键id索引取出整行，取name、city、age三个字段值，存入sort_buffer中
4. 到索引city取下一个记录的主键id
5. 重复3、4，直到city值不满足查询条件
6. 对sort_buffer中的数据按照name做快排
7. 取排序结果前1000行返回客户端

作者把这个流程称为全字段排序。

按name排序可能在内存完成，也可能需要使用磁盘临时文件辅助，取决于排序所需内存和参数`sort_buffer_size` ,这是mysql开辟的sort_buffer的大小。

以下方法确定排序语句是否用临时文件

```mysql
SET optimizer_trace='enabled=on'; /*只对本线程有效*/

select VARIABLE_VALUE into @a from performance_schema.session_status where variable_name='Innodb_rows_read';

select city,name,age from t where city='杭州' order by name limit 1000;

select * from `information_schema`.`OPTIMIZER_TRACE` \G

select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name='Innodb_rows_read';

select @b-@a;
```

通过查看OPTIMIZER_TRACE的结果确认是否用临时文件

![](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/16/1.png)

`number_of_tmp_files` 是使用的临时文件个数。内存放不下时，需要使用外部排序，一般用归并排序。mysql将需要排序的数据分成（大小适当的，我理解或许是sort_buffer的大小对应的，排序总归是要读入内存的）12份，每一份单独排序后存在这些临时文件里，然后再归并成一个有序的大文件。

`sort_buffer_size` 如果小于要排序的数据量，越小分的份数越多，`number_of_tmp_files` 越大。

`exmained_rows` 表示参与排序的行数，表中满足city=杭州的有4000行

`sort_mode` 里`packed_additional_fields` 表示排序过程对字符串做紧凑处理，即使name定义是varchar(16)，排序过程按实际长度分配空间。

在`internal_tmp_disk_storage_engine` 设置成MyISAM时，`select @b-@a` 的值是4000，表示整个执行过程只扫描4000行；`internal_tmp_disk_storage_engine` 默认是InnoDB，此时b-a的值是4001。因为查询OPTIMIZER_TRACE这个表时要用临时表，InnoDB引擎把数据从临时表取出会让`Innodb_rows_read` 加1。

### rowid排序

如果查询要返回的字段很多，sort_buffer里要放的字段数多，同样内存里能放下的行数很少，需要分很多个临时文件，排序性能会差。同理如果单行很大，性能也不好。

`max_length_for_sort_data` 是mysql控制用于排序的行数据的长度的参数，如果单行的长度超过这个值，mysql就不用全字段排序而用rowid排序（作者起的名）。

`SET max_length_for_sort_data = 16;`

city、name、age字段定义的总长度是36。新算法放入sort_buffer的字段只有要排序的列name和主键id。

执行流程变成如下：

1. 初始化sort_buffer，确定放入name和id
2. 从索引city中找到第一个满足查询条件的主键id
3. 到主键id索引取出整行，取name、id两个字段放入sort_buffer
4. 从索引city取下一个记录的主键id
5. 重复3、4直到city不满足查询条件
6. 对sort_buffer的数据按name排序
7. 遍历排序结果，取前1000行，按照id值回到原表取出city、name、age三个字段返回给客户端

mysql服务器端从排序后的sort_buffer依次取出id到原表查city、name、age的结果，不需要在服务器端耗费内存存储，直接返回给客户端。

此时`select @b-@a` 变为5000，第七步多读了1000行。

OPTIMIZER_TRACE结果中，sort_mode变为`<sort_key, rowid>` 表示参与排序的只有name和id；`number_of_tmp_files` 变为10，参与排序的行数不变，每一行变小，总数据量小了所以需要的临时文件少了。

### 全字段排序 vs rowid 排序

mysql的设计思想，如果内存够，多利用内存，尽量减少磁盘访问。

内存足够优先考虑全字段。

不是所有order by都要排序，原来的数据无序才要排序。

如果从索引取出来的主键id对应的行天然按照查询条件有序，就不用排。

`alter table t add index city_user(city, name);`

之后查询流程变成：

1. 从索引city_user找到第一个满足city='杭州'的主键id
2. 到主键id索引取出整行，取name、city、age三个字段值，作为结果的一部分直接返回
3. 到city_user取下一个记录主键id
4. 重复2、3，直到查到第1000条，或者查询条件不满足时，循环结束

此时用`explain` 语句查看，Extra里没有Using filesort，并且因为有索引，只需要读1000行

如果索引再加age字段，可以用到覆盖索引，不用回表，`explain` 语句Extra里会显示Using index表示用了覆盖索引。

注意维护索引也有代价，需要权衡。

## 17 如何正确地显示随机消息

随机从单词表里取3个单词

```mysql
CREATE TABLE `words` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `word` varchar(64) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
 
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=0;
  while i<10000 do
    insert into words(word) values(concat(char(97+(i div 1000)), char(97+(i % 1000 div 100)), char(97+(i % 100 div 10)), char(97+(i % 10))));
    set i=i+1;
  end while;
end;;
delimiter ;
 
call idata();
```

### 内存临时表

首先想到用order by rand()实现

`select word from words order by rand() limit 3;`

```mysql
mysql> explain select word from words order by rand() limit 3;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                           |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
|  1 | SIMPLE      | words | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9980 |   100.00 | Using temporary; Using filesort |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+---------------------------------+
1 row in set, 1 warning (0.01 sec)
```

Using temporary表示要用临时表，Using filesort表示需要执行排序，因此合起来是需要在临时表上排序。

对InnoDB表，会优先选择全字段排序以减少回表，减少磁盘访问；但是内存表不需要访问磁盘，优先考虑用于排序的行数据量越小越好，所以选择rowid排序。

执行流程如下：

1. 创建临时表，用memory引擎，有两个字段，第一个记做字段R，double型；第二个记做W，varchar(64)型。没有索引。
2. 从words中取出所有word，对每一个用rand()生成(0,1)的小数，(小数，word)存入(R,W)中。扫描行数10000
3. 初始化sort_buffer，buffer有两个字段，一个是double，一个是整型
4. 从内存临时表逐行取出R和位置信息pos(临时表其实是个数组，pos是数组下标)，存入buffer两个字段里。这里做全表扫描，扫描总行数变为20000
5. 在sort_buffer中按R排序
6. 排序完成取前3个结果的位置信息，到临时表中取word，返回给客户端。扫描总行数20003

#### 纠错

课件通过慢查询日志，看到`Rows_examined:20003` ，我在Mac下mysql8.0查看慢查询日志，结果是10003，跟上述执行流程不符合，不知道mysql在哪里做了优化。回看上述流程，我觉得临时表和sort_buffer的信息是冗余的，Using temporary表示用临时表，那么可能不需要再用buffer（大家都是内存），直接在临时表上排序，节省扫描10000行。

```mysql
# Time: 2021-09-09T12:23:22.767557Z
# User@Host: root[root] @ localhost []  Id:    26
# Query_time: 0.012705  Lock_time: 0.000408 Rows_sent: 3  Rows_examined: 10003
SET timestamp=1631190202;
select word from words order by rand() limit 3;
```

#### 课件随机排序完整流程图（跟我验证结果不一致）

![随机排序完整流程图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/17/1.png)

#### mysql的表定位一行数据

rowid表示的是每个引擎用来唯一标识数据行的信息。

- 有主键的InnoDB表，rowid是主键
- 没有主键的InnoDB表，rowid是系统生成的，6字节（如果没有定义主键，但是有唯一键，InnoDB会把它当主键处理）
- 用户没有显式指定主键，InnoDB会用rowid自动创建一个隐藏的主键，但是这个主键对server层透明，优化器用不上
- memory引擎不是索引组织表。在这个例子可以认为是数组，rowid就是数组下标

### 磁盘临时表

`tmp_table_size` 参数限制了内存临时表的大小，默认16M。如果临时表大小超过设置，就会转成磁盘临时表。

磁盘临时表默认用InnoDB，由参数`internal_tmp_disk_storage_engine` 控制。此时排序对应一个没有显式索引的InnoDB表排序过程。

#### 课件的结果

```mysql
set tmp_table_size=1024;
set sort_buffer_size=32768;
set max_length_for_sort_data=16;
/* 打开 optimizer_trace，只对本线程有效 */ SET optimizer_trace='enabled=on';
/* 执行语句 */
select word from words order by rand() limit 3;
/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
```

![](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/17/2.png)

`max_length_for_sort_data` 小于word字段长度定义，所以`sort_mode` 显示rowid排序，符合预期。

这个语句没有用临时文件，用的是mysql5.6引入的优先队列排序算法。

如果用归并排序，虽然也能得到结果，但是最终只需要取3个，浪费了计算量。

优先队列排序过程：

1. 对10000个准备排序的(R, rowid)，取前三行，构造一个堆。
2. 取下一行(R', rowid')，跟当前堆里最大的R比较，如果R'<R，则把(R, rowid)换成(R', rowid')
3. 重复第2步，直到第10000个(R', rowid')完成比较。

![优先队列排序算法示例](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/17/3.png)

上图模拟6行，通过优先队列排序找到最小3个R值的过程。整个过程为了最快拿到当前堆的最大值，总是保持最大值在堆顶，这是一个最大堆。

`filesort_priority_queue_optimization` 里`chosen=true` 表示用了优先队列算法。排序结束后堆里就是R最小的三行，依次把它们的rowid取出，去临时表里拿到word。

磁盘临时表是指(R, rowid)这个表临时存在磁盘，`number_of_tmp_files` 为0指排序过程中不用临时文件。

这里的临时表是InnoDB表，没有显式索引，也意味着没有主键，InnoDB会为它生成6字节的rowid自增主键（对server层透明，优化器用不上），当rowid达到最大值后，会回到最小值，覆盖之前写入的记录，因此创建表的时候都建议有主键。在源码中rowid是8字节，由于编码原因，应用中变为6字节。

上一篇文章`select city,name,age from t where city='杭州' order by name limit 1000 ;` 如果用优先队列算法，需要维护的堆是1000行的(name,rowid)，超过了设置的`sort_buffer_size` 大小，所以只能用归并排序。

不管是内存临时表还是磁盘临时表，`order by rand()` 都会让计算过程非常复杂，需要大量扫描行数，排序过程消耗资源也很大。

#### 自己实验的结果

```mysql
mysql> set tmp_table_size=1024;
Query OK, 0 rows affected (0.00 sec)

mysql> set sort_buffer_size=32768;
Query OK, 0 rows affected (0.00 sec)

mysql> set max_length_for_sort_data=16;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> SET optimizer_trace='enabled=on';
Query OK, 0 rows affected (0.00 sec)

mysql> select word from words order by rand() limit 3;
+------+
| word |
+------+
| ggaj |
| hiha |
| jdje |
+------+
3 rows in set (0.01 sec)

mysql> SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G
*************************** 1. row ***************************
                            QUERY: select word from words order by rand() limit 3
                            TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `words`.`word` AS `word` from `words` order by rand() limit 3"
          }
        ]
      }
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "substitute_generated_columns": {
            }
          },
          {
            "table_dependencies": [
              {
                "table": "`words`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          {
            "rows_estimation": [
              {
                "table": "`words`",
                "table_scan": {
                  "rows": 9980,
                  "cost": 5.25
                }
              }
            ]
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ],
                "table": "`words`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 9980,
                      "access_type": "scan",
                      "resulting_rows": 9980,
                      "cost": 1003.25,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 9980,
                "cost_for_plan": 1003.25,
                "chosen": true
              }
            ]
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": null,
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`words`",
                  "attached": null
                }
              ]
            }
          },
          {
            "optimizing_distinct_group_by_order_by": {
              "simplifying_order_by": {
                "original_clause": "rand()",
                "items": [
                  {
                    "item": "rand()"
                  }
                ],
                "resulting_clause_is_simple": false,
                "resulting_clause": "rand()"
              }
            }
          },
          {
            "finalizing_table_conditions": [
            ]
          },
          {
            "refine_plan": [
              {
                "table": "`words`"
              }
            ]
          },
          {
            "considering_tmp_tables": [
              {
                "adding_tmp_table_in_plan_at_position": 1,
                "write_method": "write_all_rows"
              },
              {
                "adding_sort_to_table": ""
              }
            ]
          }
        ]
      }
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "sorting_table": "<temporary>",
            "filesort_information": [
              {
                "direction": "asc",
                "expression": "`rand()`"
              }
            ],
            "filesort_priority_queue_optimization": {
              "limit": 3,
              "chosen": true
            },
            "filesort_execution": [
            ],
            "filesort_summary": {
              "memory_available": 32768,
              "key_size": 8,
              "row_size": 275,
              "max_rows_per_buffer": 4,
              "num_rows_estimate": 9980,
              "num_rows_found": 10000,
              "num_initial_chunks_spilled_to_disk": 0,
              "peak_memory_used": 1132,
              "sort_algorithm": "std::sort",
              "unpacked_addon_fields": "using_priority_queue",
              "sort_mode": "<fixed_sort_key, additional_fields>"
            }
          }
        ]
      }
    }
  ]
}
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0
          INSUFFICIENT_PRIVILEGES: 0
1 row in set (0.00 sec)

```

可以看到`sort_mode` 是`<fixed_sort_key, additional_fields>` ，意味着是全字段排序而不是rowid排序。原因是从mysql8.0.20起，`max_length_for_sort_data` 参数已经被抛弃，可以设置但是设置了也无效，不会影响排序。我的版本是mysql8.0.25。

因此每一行的大小允许，mysql就做了全字段排序，维护的堆应该为(R, word)，在最后得到的堆中直接返回word就好，不需要再回临时表读取。

### 如何正确地随机排序

如果只选择一个word值

#### 执行代价小但是概率可能不均的算法

1. 取表主键id的最大值M和最小值N
2. X=(M-N)*rand()+N
3. 取不小于X的第一个ID的行

```mysql
select max(id),min(id) into @M,@N from t ;
set @X= floor((@M-@N+1)*rand() + @N);
select * from t where id >= @X limit 1;
```

`max(id),min(id)` 都不需要扫描索引，`select` 可以用索引快速定位，可以认为只扫描3行。但是ID中间可能有空洞，因此选择不同行的概率不一样。比如id为1、2、40000、40001。

#### 执行代价稍大，但仍然比rand()小，概率均等的算法

1. 取整个表的行数C
2. Y=floor(C*rand())
3. 用limit Y,1 取一行

```mysql
select count(*) into @C from t;
set @Y = floor(@C * rand());
set @sql = concat("select * from t limit ", @Y, ",1"); 
prepare stmt from @sql;
execute stmt;
DEALLOCATE prepare stmt;
```

limit后不能直接跟变量，所以用了prepare+execute

mysql处理limit Y,1是按顺序读出，丢掉前Y个，再把下一个记录返回，这一步扫描Y+1行，再加上第一步C行，共C+Y+1行。

按照作者原文，order by rand的扫描行数是2C+1，只有Y随机到比较大的数时，两个算法的代价才接近，平均下来这个算法代价更小，但是按照我的实验，order by rand()的代价只有C+1。

但是另一方面，取C的时候可以选最小的索引，limit那一步可以走主键索引，即便Y=C，因为不用排序，也比rand()构建临时表然后排序的方法要快得多。

用这个算法取3个随机word的流程如下：

1. 取整表行数C
2. 用上述相同的随机方法得到Y1，Y2，Y3
3. 执行三个limit Y,1 得到3行

```mysql
select count(*) into @C from t; 
set @Y1 = floor(@C * rand());
set @Y2 = floor(@C * rand());
set @Y3 = floor(@C * rand());
select * from t limit @Y1，1; //在应用代码里面取Y1、Y2、Y3值，拼出SQL后执行 
select * from t limit @Y2，1;
select * from t limit @Y3，1;
```

## 18 为什么逻辑相同的sql语句性能差异巨大

### 对索引字段做函数操作，可能破坏索引值的有序性，导致优化器决定放弃走索引树搜索功能

```mysql
CREATE TABLE `tradelog` ( 
`id` int(11) NOT NULL,
`tradeid` varchar(32) DEFAULT NULL, 
`operator` int(11) DEFAULT NULL, 
`t_modified` datetime DEFAULT NULL, 
PRIMARY KEY (`id`),
KEY `tradeid` (`tradeid`),
KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

以下语句用不了索引的搜索功能，会走全索引扫描

`select count(*) from tradelog where month(t_modified)=7;`

不满足最左前缀，索引不起作用。

可以改成

`select count(*) from tradelog where (t_modified >= '2016-7-1' and t_modified < '2016-8-1') or (t_modified  >= '2017-7-1' and t_modified < '2017-8-1') or (t_modified >= '2018-7-1' and t_modified < '2018-8-1');`

即便是不改变有序性的函数，优化器也会偷懒不用索引。

比如`select * from tradelog where id+1=10000;` +1不会改变有序性，但是优化器还是不能用id索引快速定位到9999，所以应该写成`where id = 10000-1;` 

#### 执行过程中可能经过函数操作，但最终在拿到结果后，server层还是要做一轮判断

比如执行过程中做字符串长度截断，server层仍然要判断拿到的结果是否满足查询条件

```mysql
CREATE TABLE `table_a` (
  `id` int(11) NOT NULL,
  `b` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `b` (`b`)
) ENGINE=InnoDB;
```

假设表中有100万行，有10万行的b值是'1234567890'，执行

`select * from table_a where b='1234567890abcd';`

mysql不会看到b定义是varchar(10)就直接返回空，也不会拿'1234567890abcd'到索引树做匹配并快速判断索引树b上没有这个值返回空，而是执行很慢：

1. 传给引擎时，做了字符截断。因为引擎里b的定义长度是10，所以截前10个字节，即'1234567890'
2. 在索引树b中满足条件的有10万条记录
3. 因为是select *，做10万次回表
4. 每次回表后查出整行，到server层判断，b的值都不满足查询条件
5. 返回结果空集

### 隐式类型转换可能不能用索引

`select * from tradelog where tradeid=110717;` 需要全表扫描。

tradeid是varcher(32)，输入参数是整型，所以要做类型转换。

```mysql
mysql> select "10">9;
+--------+
| "10">9 |
+--------+
|      1 |
+--------+
1 row in set (0.00 sec)

```

在mysql中，字符串和数字做比较，是将字符串转成数字。

因此，对优化器来说，上述查询语句相当于`select * from tradelog where CAST(tradeid AS signed int) = 110717;` 触发了第一条规则，对索引字段做函数操作，优化器会放弃走树搜索功能。

但是如果对参数而不是字段做函数操作，是可以用索引的。

### 隐式字符编码转换可能不能用索引

```mysql
CREATE TABLE `trade_detail` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `trade_step` int(11) DEFAULT NULL, /* 操作步骤 */
  `step_info` varchar(32) DEFAULT NULL, /* 步骤信息 */
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
 
insert into tradelog values(1, 'aaaaaaaa', 1000, now());
insert into tradelog values(2, 'aaaaaaab', 1000, now());
insert into tradelog values(3, 'aaaaaaac', 1000, now());
 
insert into trade_detail values(1, 'aaaaaaaa', 1, 'add');
insert into trade_detail values(2, 'aaaaaaaa', 2, 'update');
insert into trade_detail values(3, 'aaaaaaaa', 3, 'commit');
insert into trade_detail values(4, 'aaaaaaab', 1, 'add');
insert into trade_detail values(5, 'aaaaaaab', 2, 'update');
insert into trade_detail values(6, 'aaaaaaab', 3, 'update again');
insert into trade_detail values(7, 'aaaaaaab', 4, 'commit');
insert into trade_detail values(8, 'aaaaaaac', 1, 'add');
insert into trade_detail values(9, 'aaaaaaac', 2, 'update');
insert into trade_detail values(10, 'aaaaaaac', 3, 'update again');
insert into trade_detail values(11, 'aaaaaaac', 4, 'commit');
```

查询id=2的交易的所有操作步骤信息可以这么写

`select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2;`

```mysql
mysql> explain select d.* from tradelog l, trade_detail d where d.tradeid=l.trad
eid and l.id=2;
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys   | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | l     | NULL       | const | PRIMARY,tradeid | PRIMARY | 4       | const |    1 |   100.00 | NULL        |
|  1 | SIMPLE      | d     | NULL       | ALL   | tradeid         | NULL    | NULL    | NULL  |   11 |    10.00 | Using where |
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-------------+
2 rows in set, 3 warnings (0.00 sec)

```

第一行显示优化器在tradelog表上查到id=2的行，这个步骤用主键索引，rows=1表示只扫描一行。

第二行key=NULL，表示没有用上trade_detail上的trade_id索引，进行全表扫描。

在这个执行计划里，从tradelog中取tradeid，再去trade_detail里查询匹配字段。把tradelog称为驱动表，trade_detail称为被驱动表，tradeid称为关联字段。

![关联语句执行流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/18/1.png)

1. 根据id在tradelog里找到L2
2. 从L2中取出tradeid字段的值
3. 根据tradeid的值到trade_detail表中查找条件匹配的行，explain的结果里面第二行key=NULL表示的就是这个过程通过遍历主键索引的方式，逐个判断tradeid的值是否匹配

把第三步单独改成sql语句

`select * from trade_detail where tradeid=$L2.tradeid.value;`

`$L2.tradeid.value` 字符集是`utf8mb4` 

`utf8mb4` 是 `utf8` 超集，当这两个编码的字符串比较时，mysql内部操作是先把utf8字符串转成utf8mb4字符串再比较

>在mysql里和程序设计语言里面，做自动类型转换的时候为了避免数据在转换过程中由于截断导致数据错误，都是“按数据长度增加的方向”转换

即这个语句等于

`select * from trade_detail where CONVERT(tradeid USING utf8mb4)=$L2.tradeid.value;`

触发了第一条规则，对索引字段做函数操作，优化器放弃走树搜索。

**连接过程中要求在被驱动表的索引字段上做函数操作，是直接导致被驱动表做全表扫描的原因。** 

查找trade_detail里id=4的操作对应的操作者

`select l.operator from tradelog l , trade_detail d where d.tradeid=l.tradeid and d.id=4;` 

```mysql
mysql> explain select l.operator from tradelog l , trade_detail d where d.tradeid=l.tradeid and d.id=4;
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys   | key     | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | d     | NULL       | const | PRIMARY,tradeid | PRIMARY | 4       | const |    1 |   100.00 | NULL                  |
|  1 | SIMPLE      | l     | NULL       | ref   | tradeid         | tradeid | 131     | const |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+-----------------+---------+---------+-------+------+----------+-----------------------+
2 rows in set, 2 warnings (0.01 sec)

mysql> show warnings\G
*************************** 1. row ***************************
  Level: Warning
   Code: 1739
Message: Cannot use ref access on index 'tradeid' due to type or collation conversion on field 'tradeid'
*************************** 2. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `crashcourse`.`l`.`operator` AS `operator` from `crashcourse`.`tradelog` `l` join `crashcourse`.`trade_detail` `d` where (('aaaaaaab' = `crashcourse`.`l`.`tradeid`))
2 rows in set (0.00 sec)

```

这里先到d查找匹配行，再取tradeid，再到l查到匹配字段，d成了驱动表。

explain第二行显示，这次查询用上了tradelog里的索引tradeid，扫描行数是1。

假设trade_detail里id=4的行为R4，被驱动表tradelog上执行的是`select operator from tradelog where tradeid=$R4.tradeid.value;` 

按照字符集转换规则，相当于`select operator from tradelog where tradeid=CONVERT($R4.tradeid.value USING utf8mb4);` ，函数操作在参数上，可以用上被驱动表的tradeid索引。

---

这里自己实验的结果跟原书不同，原书第二行Extra是NULL，同样有warning，不知道warning的内容。

从实验的warning以及google的结果，这种写法能用上索引是Using index condition即索引下推（ICP）的功劳。（但是这里没看出来跟索引下推的联系？如果按照id先到trade_detail里取tradeid再到tradelog找匹配字段，那么id已经判断过了， 用不到索引下推。）

如果把语句手动改成等价的

```mysql
mysql> explain select l.operator from tradelog l , trade_detail d where l.tradeid=CONVERT(d.tradeid USING utf8mb4) and d.id=4;
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | d     | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
|  1 | SIMPLE      | l     | NULL       | ref   | tradeid       | tradeid | 131     | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
2 rows in set, 1 warning (0.00 sec)

mysql> show warnings\G
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `crashcourse`.`l`.`operator` AS `operator` from `crashcourse`.`tradelog` `l` join `crashcourse`.`trade_detail` `d` where ((`crashcourse`.`l`.`tradeid` = convert('aaaaaaab' using utf8mb4)))
1 row in set (0.00 sec)

```

一切正常。从explain的rows结果上看原书写法也自动转换用了索引，但是warning怎么回事？是为了提示不要隐式转换字符集吗？

---

因此，要优化`select d.* from tradelog l, trade_detail d where d.tradeid=l.tradeid and l.id=2;` ，可以将trade_detail的tradeid字段也改成utf8mb4

`alter table trade_detail modify tradeid varchar(32) CHARACTER SET utf8mb4 default null;` 

但是如果数据量比较大，或业务上暂时不能做这个DDL操作，就只能修改sql语句，将参数转成utf8字符集。

## 19 为什么只查一行的语句也会执行慢？

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
 
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=100000) do
    insert into t values(i,i);
    set i=i+1;
  end while;
end;;
delimiter ;
 
call idata();
```

### 查询长时间不返回

`select * from t where id=1;`

大概率表t被锁住了。一般首先执行`show processlist` 看当前语句处于什么状态，再针对每种状态分析原因、如何复现、如何处理

#### 等MDL锁

```mysql
mysql> show processlist\G
*************************** 1. row ***************************
     Id: 5
   User: event_scheduler
   Host: localhost
     db: NULL
Command: Daemon
   Time: 3901280
  State: Waiting on empty queue
   Info: NULL
*************************** 2. row ***************************
     Id: 32
   User: root
   Host: localhost
     db: crashcourse
Command: Sleep
   Time: 115
  State: 
   Info: NULL
*************************** 3. row ***************************
     Id: 33
   User: root
   Host: localhost
     db: crashcourse
Command: Query
   Time: 40
  State: Waiting for table metadata lock
   Info: select * from t where id=1
*************************** 4. row ***************************
     Id: 35
   User: root
   Host: localhost
     db: crashcourse
Command: Query
   Time: 0
  State: init
   Info: show processlist
4 rows in set (0.00 sec)

```

出现这种状态表示有一个线程正在表t上请求或持有MDL写锁，把select堵住了。

处理方式就是找到谁持有MDL写锁，kill掉。

mysql启动时设置performance_schema=on（在mysql配置文件里修改，mysql8.0.25默认是on）

通过查询`sys.schema_table_block_waits` 可以找到造成阻塞的process id

```mysql
mysql> select blocking_pid from sys.schema_table_lock_waits;
+--------------+
| blocking_pid |
+--------------+
|           32 |
+--------------+
1 row in set (0.00 sec)

mysql> kill 32;
Query OK, 0 rows affected (0.00 sec)
```

需要`select * from t where id=1;` 正在被锁住才能查询到，否则是空集。kill掉lock table的进程后select正常执行。

#### 等flush

这种情况使用`select * from information_schema.processlist where id=...;` 可以看到STATE一列为`Waiting for table flush` 

表示有一个线程正要对t做flush操作。mysql里对表flush有

`flush tables t with read lock;`

`flush tables with read lock;`

前者只关闭t，后者关闭mysql里所有打开的表。

正常情况下这两个语句执行很快，除非它们也别的线程堵住，然后它又堵住了select

复现

|         sessionA          |     sessionB      |           sessionC            |
| :-----------------------: | :---------------: | :---------------------------: |
| `select sleep(1) from t;` |                   |                               |
|                           | `flush tables t;` |                               |
|                           |                   | `select * from t where id=1;` |

排查很简单，`show processlist` 查找id再kill掉。

#### 等行锁

`select * from t where id=1 lock in share mode;` 

这样访问id=1的记录要加读锁，如果这时候已经有事务在这行记录上持有写锁，select就会被堵住。

复现

|                     sessionA                      |                      sessionB                      |
| :-----------------------------------------------: | :------------------------------------------------: |
| `begin;`  <br /> `update t set c=c+1 where id=1;` |                                                    |
|                                                   | `select * from t where id = 1 lock in share mode;` |

在mysql8.0.25中，仍可以通过`sys.innodb_lock_waits` 表查到谁占着这个写锁

```mysql
mysql> select * from sys.innodb_lock_waits where locked_table='`crashcourse`.`t`'\G
*************************** 1. row ***************************
                wait_started: 2021-09-11 17:49:46
                    wait_age: 00:00:02
               wait_age_secs: 2
                locked_table: `crashcourse`.`t`
         locked_table_schema: crashcourse
           locked_table_name: t
      locked_table_partition: NULL
   locked_table_subpartition: NULL
                locked_index: PRIMARY
                 locked_type: RECORD
              waiting_trx_id: 421696124587336
         waiting_trx_started: 2021-09-11 17:49:46
             waiting_trx_age: 00:00:02
     waiting_trx_rows_locked: 1
   waiting_trx_rows_modified: 0
                 waiting_pid: 46
               waiting_query: select  * from t where id=1 lock in share mode
             waiting_lock_id: 140221147876680:46:5:2:140220910098976
           waiting_lock_mode: S,REC_NOT_GAP
             blocking_trx_id: 1414470
                blocking_pid: 44
              blocking_query: NULL
            blocking_lock_id: 140221147878360:46:5:2:140220910108192
          blocking_lock_mode: X,REC_NOT_GAP
        blocking_trx_started: 2021-09-11 17:46:32
            blocking_trx_age: 00:03:16
    blocking_trx_rows_locked: 1
  blocking_trx_rows_modified: 1
     sql_kill_blocking_query: KILL QUERY 44
sql_kill_blocking_connection: KILL 44
1 row in set (0.00 sec)

mysql> kill 44;
Query OK, 0 rows affected (0.00 sec)
```

把blocking_pid kill掉即可。

kill query是停止44号线程当前正在执行的语句，这个方法是没用的，因为占有行锁的是update语句，已经执行完了。现在kill query无法让这个事务去掉id=1上的行锁。实际上只有kill 44才有效，即直接断开对应的连接。隐含的逻辑是连接被断开的时候，回自动回滚这个连接里正在执行的线程，也就释放了id=1上的行锁。

### 查询慢

坏查询不一定是慢查询，但是当业务量变大时，坏查询会变成慢查询。

只扫描一行，但是很慢的语句：

`select * from t where id=1;`

复现过程:

|                     sessionA                     |                 sessionB                 |
| :----------------------------------------------: | :--------------------------------------: |
|  `start transaction with consistent snapshot;`   |                                          |
|                                                  | `update t set c=c+1 where id=1;//100w次` |
|          `select * from t where id=1;`           |                                          |
| `select * from t where id=1 lock in share mode;` |                                          |

sessionA:

```mysql
mysql> start transaction with consistent snapshot;
Query OK, 0 rows affected (0.00 sec)

mysql> set global slow_query_log=on;
Query OK, 0 rows affected (0.01 sec)

mysql> set long_query_time=0;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where id=1;
+----+------+
| id | c    |
+----+------+
|  1 |    2 |
+----+------+
1 row in set (1.24 sec)

mysql> select * from t where id=1 lock in share mode;
+----+---------+
| id | c       |
+----+---------+
|  1 | 1000002 |
+----+---------+
1 row in set (0.00 sec)
```

sessionB:

```mysql
mysql> delimiter ;;
mysql> create procedure idata()
    -> begin
    -> declare i int;
    -> set i=1;
    -> while(i<=1000000) do
    -> update t set c=c+1 where id=1;
    -> set i=i+1;
    -> end while;
    -> end;;
Query OK, 0 rows affected (0.00 sec)

mysql> delimiter ;
mysql> call idata();
Query OK, 1 row affected (1 min 45.62 sec)
```

slow_query_log:

```mysql
# Time: 2021-09-12T07:41:03.017489Z
# User@Host: root[root] @ localhost []  Id:    46
# Query_time: 1.238196  Lock_time: 0.000869 Rows_sent: 1  Rows_examined: 1
SET timestamp=1631432461;
select * from t where id=1;
# Time: 2021-09-12T07:41:18.523615Z
# User@Host: root[root] @ localhost []  Id:    46
# Query_time: 0.002541  Lock_time: 0.000430 Rows_sent: 1  Rows_examined: 1
SET timestamp=1631432478;
select * from t where id=1 lock in share mode;

```

sessionB更新完100w次，生成100w个undo log

带`lock in share mode` 的sql语句是当前读，因此会直接读到最后的结果，所以很快。而不带的是一致性读，需要从最新版本的值开始依次执行undo log，100万次后才将2返回。

undo log里记的是“把3改成2”、“把4改成3”这样的操作逻辑。

## 20 幻读

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;
 
insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

InnoDB默认事务隔离级别是可重复读，本文接下来没有特殊说明都默认这个级别。

### 幻读是什么，有什么问题

```mysql
begin;
select * from t where d=5 for update;
commit;
```

d上没有索引，所以这条语句会走主键索引做全表扫描。d=5这一行对应id=5，是一定会加写锁做当前读的，并且会在commit执行的时候释放。在读提交下，select执行完成后，只有行锁，并且InnoDB会把不满足条件的行行锁去掉，在执行commit时，释放d=5这一行的行写锁。那么在可重复读下，其他被扫描到的，但是不满足d=5的记录会不会加锁？

如果只在id=5这一行加锁，而其他行不加锁：

**假设** 以下场景:

|      |                           sessionA                           |            sessionB            | sessionC                       |
| :--: | :----------------------------------------------------------: | :----------------------------: | ------------------------------ |
|  T1  | `begin;`  <br /> `select * from t where d=5 for update;/*Q1*/` <br /> `result:(5,5,5)` |                                |                                |
|  T2  |                                                              | `update t set d=5 where id=0;` |                                |
|  T3  | `select * from t where d=5 for update;/*Q2*/` <br /> `result:(0,0,5),(5,5,5)` |                                |                                |
|  T4  |                                                              |                                | `insert into t values(1,1,5);` |
|  T5  | `select * from t where d=5 for update;/*Q3*/` <br /> `result:(0,0,5),(1,1,5),(5,5,5)` |                                |                                |
|  T6  |                          `commit;`                           |                                |                                |

Q3读到id=1这一行的现象被称为幻读。幻读指的是一个事务在前后两次查询同一个范围（同一个查询条件）的时候，后一次查询看到了前一次查询没有看到的行。

1. 可重复读隔离级别下，普通的查询是快照读，不会看到别的事务插入的数据。因此，幻读在“当前读”下才会出现
2. sessionB的结果被sessionA用当前读看到，不能称为幻读。幻读仅专指新插入的行

三个查询都是加了`for update` ，都是当前读。当前读就是要读到所有已经提交的记录的最新值。并且sessionBC的语句执行后就会提交，跟事务的可见性规则不矛盾。

#### 幻读有什么问题？

首先是语义问题。sessionA在T1声明“锁住d=5的所有行，不准别的事务进行读写操作”，这个语义被破坏了。

为了看更明显，假设以下场景

|      |                           sessionA                           |                           sessionB                           |                           sessionC                           |
| :--: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  T1  | `begin;`  <br /> `select * from t where d=5 for update;/*Q1*/` |                                                              |                                                              |
|  T2  |                                                              | `update t set d=5 where id=0;` <br /> `update t set c=5 where id=0;` |                                                              |
|  T3  |        `select * from t where d=5 for update;/*Q2*/`         |                                                              |                                                              |
|  T4  |                                                              |                                                              | `insert into t values(1,1,5);` <br />  `update t set c=5 where id=1;` |
|  T5  |        `select * from t where d=5 for update;/*Q3*/`         |                                                              |                                                              |
|  T6  |                          `commit;`                           |                                                              |                                                              |

sessionB和sessionC对“id=0，d=5”这一行和“id=1，d=5”这一行的修改，破坏了Q1的加锁声明的语义。

其次，是数据一致性问题。

我们知道锁的设计是为了保证数据的一致性，这个一致性不止是数据库内部数据状态在此刻的一致性，还包括数据和日志在逻辑上的一致性。

再假设以下场景：

|      |                           sessionA                           |                           sessionB                           |                           sessionC                           |
| :--: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  T1  | `begin;`  <br /> `select * from t where d=5 for update;/*Q1*/` <br /> `update t set d=100 where d=5;` |                                                              |                                                              |
|  T2  |                                                              | `update t set d=5 where id=0;` <br /> `update t set c=5 where id=0;` |                                                              |
|  T3  |        `select * from t where d=5 for update;/*Q2*/`         |                                                              |                                                              |
|  T4  |                                                              |                                                              | `insert into t values(1,1,5);` <br />  `update t set c=5 where id=1;` |
|  T5  |        `select * from t where d=5 for update;/*Q3*/`         |                                                              |                                                              |
|  T6  |                          `commit;`                           |                                                              |                                                              |

sessionA在T1时刻新加了update语句。update的加锁语义和`select ... for update` 是一致的。

数据库里，记录是(0,5,5),(1,5,5),(5,5,100)

binlog里面的内容：

1. T2，sessionB事务提交，写入两条语句
2. T4，sessiocC事务提交，写入两条语句
3. T6，sessionA事务提交，写入`update t set d=100 where d=5;`

```mysql
update t set d=5 where id=0;
update t set c=5 where id=0;

insert into t values(1,1,5);
update t set c=5 where id=1;

update t set d=100 where d=5;
```

这个语句序列，执行之后会变成(0,5,100),(1,5,100),(5,5,100)。

即不管拿到备库执行，还是用binlog克隆一个库，都会和原库数据不一致。

这个不一致是假设`select * from t where d=5 for update;` 只给d=5的行加写锁导致的。

**假设把全表扫描过程中碰到的行都加上写锁** 

|      |                           sessionA                           |                           sessionB                           |                           sessionC                           |
| :--: | :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  T1  | `begin;`  <br /> `select * from t where d=5 for update;/*Q1*/` <br /> `update t set d=100 where d=5;` |                                                              |                                                              |
|  T2  |                                                              | `update t set d=5 where id=0(blocked);` <br /> `update t set c=5 where id=0;` |                                                              |
|  T3  |        `select * from t where d=5 for update;/*Q2*/`         |                                                              |                                                              |
|  T4  |                                                              |                                                              | `insert into t values(1,1,5);` <br />  `update t set c=5 where id=1;` |
|  T5  |        `select * from t where d=5 for update;/*Q3*/`         |                                                              |                                                              |
|  T6  |                          `commit;`                           |                                                              |                                                              |

sessionB在第一个update语句被锁住，T6sessionA提交后才能继续执行。

数据库里记录是(0,5,5),(1,5,5),(5,5,100)

binlog里序列是:

```mysql
insert into t values(1,1,5);
update t set c=5 where id=1;

update t set d=100 where d=5;

update t set d=5 where id=0;
update t set c=5 where id=0;
```

记录是(0,5,5),(1,5,100),(5,5,100)。幻读仍未解决。

在T3，给所有行加写锁时，id=1这一行不存在，加不上锁。

### InnoDB如何解决幻读

产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。为了解决幻读，InnoDB引入间隙锁（Gap Lock）。

间隙锁，锁的就是两个值之间的空隙，表t初始化插入了6个记录，产生了7个间隙。

下图表示主键索引上的行锁和间隙锁

![表t主键索引上的行锁和间隙锁](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/20/1.png)

当执行`select * from t where d=5 for update` 时不止给6个记录加行写锁，还加了7个间隙锁。确保无法插入新的记录。

行锁分为读锁和写锁，读锁和读锁兼容，写锁和其他行锁冲突。

跟行锁有冲突的是另外一个行锁。跟间隙锁有冲突的是插入操作，间隙锁之间不存在冲突关系。

|                           sessionA                           |                        sessionB                         |
| :----------------------------------------------------------: | :-----------------------------------------------------: |
| `begin;` <br /> `select * from t where c=7 lock in share mode` |                                                         |
|                                                              | `begin;` <br /> `select * from t where c=7 for update;` |

表里没有c=7的记录，A和B都加间隙锁，B不会被阻塞。

间隙锁和行锁合成next-key lock，每个next-key lock是前开后闭区间。表t初始化后，用`select * from t for update;` 把所有记录锁起来，形成7个next-key lock：$$(-\infin,0],(0,5],(5,10]...(25,+supremum]$$ 

因为$$+\infin$$ 是开区间，实际上，InnoDB给每个索引都加了一个不存在的最大值supremum。

next-key lock的引入解决了幻读，但同时可能带来死锁。

假设有以下业务逻辑：

```mysql
begin;
select * from t where id=N for update;

/*如果行不存在*/
insert into t values(N,N,N);

/*如果行存在*/
update t set d=N where id=N;

commit;
```

这个逻辑一旦并发，就会死锁。假设N=9

|                           sessionA                           |                         sessionB                         |
| :----------------------------------------------------------: | :------------------------------------------------------: |
|   `begin;` <br /> `select * from t where id=9 for update;`   |                                                          |
|                                                              | `begin;` <br />``select * from t where id=9 for update;` |
|                                                              |         `insert into t values(9,9,9);(blocked)`          |
| `insert into t values(9,9,9);(ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction)` |                                                          |

前两个`select ... for update` 会分别加间隙锁，不会冲突，B的插入被A的间隙锁挡住，等待；A的插入被B的间隙锁挡住，死锁。

InnoDB的死锁检测马上发现死锁关系，令A的insert报错返回。

**间隙锁的引入可能会导致同样的语句锁住更大范围，影响并发度。** 

间隙锁在可重复读下才会生效，如果是读提交，就没有间隙锁了。同时需要解决可能出现的数据和日志不一致问题，需要把binlog格式设置为row。这是不少公司的配置组合。

## 21 为什么只改一行的语句，锁这么多？

**原文写作时mysql 5.x系列最新版5.7.24，8.0系列8.0.13，版本超过这两个的，规则可能不适用。**  ~~这里应该就是原作者全文对应的最新版了，难怪有些内容不对？~~ 

间隙锁在可重复读下才有效，所以本文接下来默认可重复读。

在读提交下，语句执行过程中加上的行锁，在语句执行完成后，不满足条件的行上的行锁直接释放，不需要等到事务提交。

两个“原则”、两个“优化”、一个“bug”：

1. 原则1:加锁的基本单位是next-key lock，前开后闭。
2. 原则2:查找过程中访问到的对象才会加锁。
3. 优化1:索引上的等值查询，给唯一索引加锁的时候，next-key lock退化为行锁。
4. 优化2:索引上的等值查询，向右遍历且最后一个值不满足等值条件的时候，next-key lock退化为间隙锁。
5. bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。（8.0.25版本实验似乎已经修复）

仍以章节20的表t为例。

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;
 
insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

### 等值查询间隙锁

|                     sessionA                     |                sessionB                 |               sessionC                |
| :----------------------------------------------: | :-------------------------------------: | :-----------------------------------: |
| `begin;` <br /> `update t set d=d+1 where id=7;` |                                         |                                       |
|                                                  | `insert into t values(8,8,8);(blocked)` |                                       |
|                                                  |                                         | `update t set d=d+1 where id=10;(OK)` |

表中没有id=7。

1. 加锁单位是next-key lock，A的加锁范围是(5,10]
2. 根据优化2，退化成间隙锁，锁(5,10)

所以B被锁住，C可以。

### 非唯一索引等值锁

关于覆盖索引上的锁：

|                           sessionA                           |               sessionB               |                sessionC                 |
| :----------------------------------------------------------: | :----------------------------------: | :-------------------------------------: |
| `begin;` <br /> `select id from t where c=5 lock in share mode;` |                                      |                                         |
|                                                              | `update t set d=d+1 where id=5;(OK)` |                                         |
|                                                              |                                      | `insert into t values(7,7,7);(blocked)` |

A要给索引c上c=5这一行加读锁。

1. 加锁单位是next-key lock，因此给(0,5]上锁
2. c是普通索引，仅访问c=5这一条记录不能马上停下，需要向右遍历查到c=10才停止。根据原则2，访问到的都要加锁，因此给(5,10]加next-key lock
3. 根据优化2，(5,10]的锁退化成间隙锁(5,10)
4. 根据原则2，只有访问到的对象才会加锁。查询使用覆盖索引，不需要访问主键索引，所以主键索引上没有加锁，因此B可以完成，C会被间隙锁阻塞

`lock in share mode` 只锁覆盖索引，`for update` 系统会认为接下来要更新数据，因此会给主键索引上满足条件的行也加锁。

这个例子说明，**锁是加在索引上的** 。

如果要用`lock in share mode` 给行加读锁来避免数据被更新，必须绕过覆盖索引的优化，在查询返回字段中加入索引中不存在的字段。

### 主键索引范围锁

`select * from t where id=10 for update;`

`select * from t where id >= 10 and id < 11 for update;`

逻辑上两条语句等价，但是加锁规则不一样。

|                           sessionA                           |                           sessionB                           |                  sessionC                  |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :----------------------------------------: |
| `begin;` <br /> `select * from t where id>=10 and id<11 for update;` |                                                              |                                            |
|                                                              | `insert into t values(8,8,8);` (OK)<br /> `insert into t values(13,13,13);(blocked)` |                                            |
|                                                              |                                                              | `update t set d=d+1 where id=15;(blocked)` |

1.  开始执行时，要找到第一个id=10的行，因此本该是next-key lock(5,10]，根据优化1，退化成行锁，只加了id=10这一行的行锁。
2. 在主键索引上向右遍历，到id=15这一行停止，因此加next-key lock(10,15]

A加锁的范围是主键索引上行锁id=10和next-key lock(10,15]。

首次A定位查找id=10的行时，是当作等值查询来判断，而向右扫描到id=15时，是范围查询判断。

#### 纠错

在mysql8.0.25实验，C的语句执行成功。怀疑(10,15]退化成间隙锁。但是下文案例中15也被锁住，说明非唯一索引的next-key lock并没有退化成间隙锁。

唯一索引上范围查询也适用优化2？

从下文“唯一索引范围锁bug”里自己做的实验看，所谓的“一个bug”应该已经被修复了。但是那个案例里id<=15的条件可以让索引停止扫描，不扫到id=20，在这里必须要扫描到id=15的那一行，按理说是要上next-key lock(10,15]的，插入13失败说明间隙锁生效，只能认为在8.0.25版本，唯一索引上的范围查询也适用优化2。

### 非唯一索引范围锁

|                           sessionA                           |                sessionB                 |                 sessionC                  |
| :----------------------------------------------------------: | :-------------------------------------: | :---------------------------------------: |
| `begin;` <br /> `select * from t where c>=10 and c<11 for update;` |                                         |                                           |
|                                                              | `insert into t values(9,9,9);(blocked)` |                                           |
|                                                              |                                         | `update t set d=d+1 where c=15;(blocked)` |

第一次用c=10定位记录时，加上了next-key lock(5,10]，由于c是非唯一索引，不能优化，所以A锁的是(5,10]和(10,15]两个next-key lock。

InnoDB要扫到c=15，才知道不需要继续遍历。

在sessionC中执行`update t set d=d+1 where id=15;` 成功，而sessionA是select *，也是要回表锁主键索引的，所以这里是索引c在判断c不满足条件就不回表了所以不锁主键索引？

### 唯一索引范围锁bug

**注意版本，以下内容在8.0.25中已失效。**

|                           sessionA                           |                  sessionB                  |                  sessionC                  |
| :----------------------------------------------------------: | :----------------------------------------: | :----------------------------------------: |
| `begin;` <br /> `select * from t where id>10 and id<=15 for update;` |                                            |                                            |
|                                                              | `update t set d=d+1 where id=20;(blocked)` |                                            |
|                                                              |                                            | `insert into t values(16,16,16);(blocked)` |

按原则1和2，主键索引上应该只有next-key lock(10,15]，并且id是唯一键，所以判断到id=15这一行就应该停止了。

但是实现上，InnoDB会扫到第一个不满足条件的行id=20为止。由于是范围查询，所以索引id上(15,20]这个next-key lock也会锁上。

#### 纠错

在8.0.25中，B和C都顺利执行，所谓的bug应该已经被修复了。

### 非唯一索引上存在“等值”的例子

`insert into t values(30,10,30);`

索引c：

| c    | 0    | 5    | 10   | 10   | 15   | 20   | 25   |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| id   | 0    | 5    | 10   | 30   | 15   | 20   | 25   |

c=10的两个记录主键值不同，因此这两条记录之间也有间隙。

delete的加锁逻辑跟`select...for update` 类似

|                  sessionA                   |                  sessionB                  |               sessionC               |
| :-----------------------------------------: | :----------------------------------------: | :----------------------------------: |
| `begin;` <br /> `delete from t where c=10;` |                                            |                                      |
|                                             | `insert into t values(12,12,12);(blocked)` |                                      |
|                                             |                                            | `update t set d=d+1 where c=15;(OK)` |

A遍历时先访问第一个c=10的记录，根据原则1，加(c=5,id=5)到(c=10,id=10)这个next-key lock

然后A向右遍历，直到碰到(c=15,id=15)这一行循环结束。根据优化2，这是等值查询，向右查找遇到不满足条件的行，退化成(c=10,id=10)到(c=15,id=15)的间隙锁。

![delete加锁效果示例](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/21/1.png)

加锁范围是蓝色部分，虚线代表开区间。

### limit语句加锁

|                      sessionA                       |               sessionB                |
| :-------------------------------------------------: | :-----------------------------------: |
| `begin;` <br /> `delete from t where c=10 limit 2;` |                                       |
|                                                     | `insert into t values(12,12,12)(OK);` |

从逻辑上加不加limit 2都一样，但是加锁效果不一样。

加了limit 2的限制，在遍历到(c=10,id=30)这一行之后，满足条件的语句已经有2条，循环结束。

因此，加锁范围变成了(c=5,id=5)到(c=10,id=30)的前开后闭区间。

**在删除数据时尽量加limit** ，不仅更安全，还减小加锁范围。

### 死锁例子

|                           sessionA                           |                           sessionB                           |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| `begin;` <br /> `select id from t where c=10 lock in share mode;` |                                                              |
|                                                              |          `update t set d=d+1 where c=10;(blocked)`           |
|              `insert into t values(8,8,8);(OK)`              |                                                              |
|                                                              | `ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction` |

1. A在索引c上加next-key lock (5,10]和间隙锁(10,15)
2. B的update也要在索引c上加next-key lock (5, 10]，进入锁等待
3. A要插入，被B的间隙锁锁住，出现死锁，InnoDB让B回滚。

B的加锁操作分为两步，先加(5,10)的间隙锁，成功，然后加c=10的行锁，这时候才被阻塞，因为在这里就被阻塞，所以后面的间隙锁(10,15)B就加不上去了。

加锁具体执行的时候，是分成间隙锁和行锁两段执行的。

### order by ... desc造成的改变

|                           sessionA                           |                sessionB                 |
| :----------------------------------------------------------: | :-------------------------------------: |
| `begin;` <br /> `select * from t where c>=15 and c<=20 order by c desc lock in share mode;` |                                         |
|                                                              | `insert into t values(6,6,6);(blocked)` |

1. 由于是`order by c desc` ，第一个要定位的是索引c上“最右边的”c=20的行，所以会上间隙锁(20,25)和next-key lock (15,20]。（这里我的理解是，c=20用等值查询，非唯一索引要扫到c=25才停止，所以锁(15,20]和(20,25]，根据优化2，(20,25]退化成间隙锁）
2. 在索引c上向左遍历，要扫到c=10才停止，所以next-key lock会加到(5,10]，因为是向左遍历，所以没有优化。这正是B阻塞的原因。
3. 在扫描过程中，c=20，15这两行都存在值，由于是`select *` ，所以会在主键id索引上加两个行锁。（原文还说在(c=10,id=10)的行上也加行锁，但是经过验证`update ... id=10` 执行成功，说明c=10对应的行并没有上锁。我的理解是在索引c上锁了c=10，但是因为不满足查询条件，就没有回表，所以没有锁id=10。）

因此，A的语句锁的范围是索引c上(5,25)，主键索引上id=15，id=20两条记录的行读锁。

## 22 mysql有哪些可能存在风险的提高性能的临时方案

### 短连接风暴

正常的短连接模式就是连接到数据库执行很少的sql就断开，下次需要再重连。短连接在业务高峰期可能出现连接数暴涨。

mysql建立连接除了正常的网络连接三次握手，还有登录权限判断、获得这个连接的数据读写权限，成本很高。

短连接模型一旦数据库处理慢一些，连接数就会暴涨。

`max_connections` 参数控制一个mysql实例同时存在的连接数上限，超过这个值系统会拒绝接下来的连接请求。

机器负载高时，处理现有请求时间变长，每个连接保持的时间更长，这时再有新连接可能就会超过`max_connections` 限制。

如果调高`max_connections` ，系统负载可能进一步加大，大量资源耗费在权限验证等逻辑上，结果可能已经连接的线程拿不到cpu去执行业务的sql请求。

两种有损解决方案：

#### 先处理占着连接但不工作的线程

对不需要保持的连接，可以通过kill connection主动踢掉。这个 行为跟设置`wait_timeout` 效果一样。`wait_timeout` 表示线程空闲这么多秒后，会被mysql断开连接。

在`show processlist` 结果里踢掉sleep的线程可能有损。

|       |                    sessionA                    |           sessionB            |       sessionC       |
| :---: | :--------------------------------------------: | :---------------------------: | :------------------: |
|   T   | `begin;` <br /> `insert into t values(1,1,1);` | `select * from t where id=1;` |                      |
| T+30s |                                                |                               | `show process list;` |

A没提交，所以断开A，mysql只能回滚事务；而断开B没什么大影响。所以，优先断开像B这样事务外空闲的连接。

```mysql
mysql> show processlist;
+----+-----------------+-----------+-------------+---------+--------+------------------------+------------------+
| Id | User            | Host      | db          | Command | Time   | State                  | Info             |
+----+-----------------+-----------+-------------+---------+--------+------------------------+------------------+
|  5 | event_scheduler | localhost | NULL        | Daemon  | 222100 | Waiting on empty queue | NULL             |
| 38 | root            | localhost | crashcourse | Sleep   |    125 |                        | NULL             |
| 39 | root            | localhost | crashcourse | Sleep   |    114 |                        | NULL             |
| 60 | root            | localhost | crashcourse | Query   |      0 | init                   | show processlist |
+----+-----------------+-----------+-------------+---------+--------+------------------------+------------------+
4 rows in set (0.01 sec)
```

id=38和id=39都是sleep状态，要看事务具体状态，查看`information_schema.innodb_trx` 表

```mysql
mysql> select * from information_schema.innodb_trx\G
*************************** 1. row ***************************
                    trx_id: 3415265
                 trx_state: RUNNING
               trx_started: 2021-09-15 18:21:08
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 2
       trx_mysql_thread_id: 38
                 trx_query: NULL
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 1
          trx_lock_structs: 1
     trx_lock_memory_bytes: 1136
           trx_rows_locked: 0
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 0
          trx_is_read_only: 0
trx_autocommit_non_locking: 0
       trx_schedule_weight: NULL
1 row in set (0.00 sec)
```

`trx_mysql_thread_id: 38` 表示id=38的线程还处在事务中。

因此如果连接数过多，优先断开事务外空闲太久的连接；还不够再考虑断开事务内空闲太久的连接。

从服务端断开连接用`kill connection id` ，一个sleep的客户端连接被断开后，直到它发起下一个请求，才会收到报错`ERROR 2013 (HY000): Lost connection to MySQL server during query`

从数据库端主动断开连接可能有损，有的应用端收到错误不重连而是用原先已经不能用的句柄重试查询，从应用端看就是mysql一直没恢复。因此即使只是一个断开连接的操作，也要确保通知到业务开发团队。

#### 减少连接过程的消耗

如果数据库确认是被大量的连接数打挂了，一种可能的做法是，让数据库跳过权限验证阶段。

方法是重启数据库，并用`-skip-grant-tables` 参数启动。mysql会跳过所有权限验证阶段，包括连接过程和语句执行过程，风险极高。

尤其是库外网可访问，更不能这么做。

在mysql8.0，如果启动这个参数，mysql默认把`-skip-networking` 打开，表示这时候数据库只能被本地客户端连接。

### 慢查询性能问题

#### 索引没设计好

一般通过紧急创建索引解决。mysql5.6后，创建索引都支持Online DDL了（那前文提到过的全文索引和空间索引呢？）。这种情况最高效就是执行`alter table` 语句

比较理想的是在备库先执行。假设主库A，备库B

1. B上执行`set sql_log_bin=off;` ，不写binlog，然后`alter table` 加索引
2. 执行主备切换
3. 这时主B备A，在A上执行`set sql_log_bin=off;` ，然后`alter table` 加索引

紧急处理时，这个方案效率最高。

#### sql语句没写好

例如出现第18章的错误导致没用上索引。

在来不及修改sql语句时，mysql5.7提供query_rewrite功能，把输入的一种语句改写成另一种模式。

通过`call query_rewrite.flush_rewrite_rules()` 让新规则生效。通过`show warnings` 确认规则是否生效。

#### mysql选错索引

应急方案就是自己加或者用query_rewrite给语句加上`force index` 。

## 23 mysql怎么保证数据不丢

### binlog的写入机制

事务执行过程中，先把日志写binlog cache，提交的时候，再把binlog cache写到binlog文件。

一个事务的binlog不能拆开，不论事务多大，也要保证一次性写入。

系统给每个线程的binlog cache分配一片内存，参数`binlog_cache_size` 控制单个线程binlog cache的大小，如果超过就要暂存磁盘。

![binlog写盘状态](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/23/1.png)

每个线程有自己的binlog cache，共用同一份binlog文件。write指把日志写到文件系统的page cache，并没有持久化。fsync才是持久化，一般认为fsync才占磁盘IOPS。

write和fsync时机由`sync_binlog` 控制：

1. =0时，每次提交事务都只write，不fsync
2. =1时，每次提交事务都fsync
3. =N(N>1)时，每次提交事务都write，累积N个事务再fsync

常见将其设置成[100, 1000]。

### redo log的写入机制

redo log有三种状态：

1. 存在redo log buffer中，物理上是mysql进程内存中
2. 写到磁盘（write）但是没有持久化（fsync），物理上是在文件系统的page cache里
3. 持久化到磁盘hard disk

InnoDB提供`innodb_flush_log_at_trx_commit` 参数控制redo log的写入策略，有三种可能值：

1. =0时每次事务提交时都只把redo log留在redo log buffer中
2. =1时每次事务提交时都将redo log直接持久化到磁盘
3. =2时每次事务提交时都只把redo log 写到page cache

InnoDB的后台线程每隔一秒就会把redo log buffer中的日志调用write写到page cache，然后调用fsync持久化。

事务可能有很多语句，执行过程中的redo log也是直接写到redo log buffer，这些也会被一起持久化。即没有提交的事务的redo log也可能已经持久化到磁盘。

除了后台线程每秒一次的轮询，还有两种场景让没有提交的事务的redo log持久化：

1. redo log buffer占用的空间即将达到`innodb_log_buffer_size` 一半，后台线程会主动写盘。这里只是write，没有调用fsync
2. 并行的事务提交时顺带将当前事务的redo log buffer持久化。假设事务A执行到一半，已经写了一些redo log到buffer，B提交，`innodb_flush_log_at_trx_commit` 为1，B要把buffer里的日志全部持久化，会带上A的buffer里的日志一起持久化。

`innodb_flush_log_at_trx_commit` 设成1，在两阶段提交中，redo log在prepare阶段就要持久化一次。因为有一个崩溃恢复逻辑依赖prepare的redo log和binlog，见第15章。

每秒一次的后台轮询刷盘加上这个崩溃恢复逻辑，InnoDB认为redo log在commit时不需要fsync，write到page cache就足够。

### 组提交（group commit）

LSN（log sequence number），日志逻辑序列号，单调递增，对应redo log的一个个写入点。每次写入长度length的redo log，LSN的值就会加上length。

![redo log group commit](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/23/2.png)

trx1、trx2、trx3三个并发事务都写完redo log buffer，都在prepare阶段，持久化的过程中，对应的LSN为50，120，160

1. trx1第一个到达（第一个处于prepare？），被选为组leader
2. 等trx1要开始写盘，组里已经有3个事务，LSN变成了160
3. trx1写盘时带的LSN=160，trx1返回时所有LSN<=160的redo log都已经持久化
4. trx2和trx3直接返回

并发更新场景下，第一个事务写完redo log buffer后，接下来这个fsync调用越晚，组员就可能越多，节约磁盘IOPS效果越好。

两阶段提交实际上会再细化。

![两阶段提交示意图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/15/1.png)

写binlog分成两步：

1. 先把binlog 从binlog cache写到页缓存
2. 调用fsync持久化

mysql做了“拖时间”的优化，让组提交效果更好，把redo log做fsync的时间拖到上述步骤1之后，使redo log和binlog都可以组提交。

![两阶段提交细化](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/23/3.png)

第4步binlog的fsync也是组提交，减少磁盘IOPS消耗。

可以设置`binlog_group_commit_sync_delay` 和`binlog_group_commit_sync_no_delay_count` 提升binlog组提交效果。

前者表示延迟多少微秒才fsync，后者表示累积多少次才调用fsync，两者是或的关系，满足其一就调用fsync。

这两个参数的逻辑在`sync_binlog` 之前，即使设置`sync_binlog` 为0，组提交该等的还是会等，满足了两个条件之一，进入`sync_binlog` 阶段，如果判断为0，直接跳过，不`fsync` 。

WAL机制减少磁盘写，但是每次提交事务都要写redo log和binlog，磁盘读写次数看起来没变少，但WAL得益于两方面：

1. redo log 和 binlog都是顺序写，磁盘顺序写比随机写速度要快（但是对于固态硬盘应该没有区别才对？）
2. 组提交机制可以大幅降低磁盘的IOPS消耗。

### mysql在IO上出现性能瓶颈怎么提升性能



1. 设置`binlog_group_commit_sync_delay` 和`binlog_group_commit_sync_no_delay_count` 减少binlog写盘次数。这个方法基于“额外的故意等待”，可能增加语句响应时间，但是没有丢数据风险
2. `sync_binlog` 设成大于1，常见[100,1000]。风险是主机掉电会丢binlog
3. 将`innodb_flush_log_at_trx_commit`  设为2，风险是主机掉电时会丢数据。不建议设0，如果设0，redo log只存在于内存，mysql异常重启也会丢数据。设2和0性能差不多，风险更小

### 小结

 第2章、第15章分析redo log和binlog完整如何保证crash-safe，本章分析怎么保证redo log和binlog完整。

## 24 mysql怎么保证主备一致

### 主备切换和同步流程

基本的主备切换流程

![mysql基本主备切换流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/24/1.png)

备库设readonly可以防止运营查询误操作、防止切换逻辑有bug造成主备不一致、判断节点角色。

readonly对super权限用户无效，负责主备同步更新的线程拥有超级权限。

下图是一个update在A执行，同步到B的完整流程。

![mysql主备同步流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/24/2.png)

备库B和主库A之间维持一个长连接，A内部有一个线程专门服务这个长连接。一个事务日志同步的完整过程：

1. B通过`change master` 命令，设置A的IP、端口、用户名、密码、从哪个位置开始请求binlog，这个位置包含文件名和日志偏移量
2. B执行`start slave` 命令，启动两个线程，`io_thread` 和`sql_thread` ，`io_thread`  负责和A建立连接
3. A校验完用户名、密码后，按照位置从本地读binlog，发给B
4. B拿到binlog后写到本地文件，称为中转日志(relay log)
5. `sql_thread` 读取relay log，解析出命令，并执行

目前`sql_thread` 已经演化成多个线程。

### binlog的三种格式对比

三种格式，statement，row，mixed，即前两种混合。

查看当前正在写入的binlog文件`show master status;`

查看指定binlog内容`show binlog events in 'binlog文件名';`

查看和设定binlog格式等信息`show variables like '%binlog%';` `set binlog_format=statement;`

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `a` (`a`),
  KEY `t_modified`(`t_modified`)
) ENGINE=InnoDB;
 
insert into t values(1,1,'2018-11-13');
insert into t values(2,2,'2018-11-12');
insert into t values(3,3,'2018-11-11');
insert into t values(4,4,'2018-11-10');
insert into t values(5,5,'2018-11-09');
```

执行`delete from t /*comment*/ where a>=4 and t_modified<='2018-11-10' limit 1;`

#### statement格式

当binlog_format=statement时，binlog记录的就是sql原文

```mysql
mysql> delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit
1;
Query OK, 1 row affected, 1 warning (0.00 sec)

mysql> show binlog events in 'binlog.000007';
+---------------+-----+----------------+-----------+-------------+-----------------------------------------------------------------------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                                                                                          |
+---------------+-----+----------------+-----------+-------------+-----------------------------------------------------------------------------------------------+
| binlog.000007 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.25, Binlog ver: 4                                                             |
| binlog.000007 | 125 | Previous_gtids |         1 |         156 |                                                                                               |
| binlog.000007 | 156 | Anonymous_Gtid |         1 |         235 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                          |
| binlog.000007 | 235 | Query          |         1 |         339 | BEGIN                                                                                         |
| binlog.000007 | 339 | Query          |         1 |         512 | use `crashcourse`; delete from t /*comment*/  where a>=4 and t_modified<='2018-11-10' limit 1 |
| binlog.000007 | 512 | Xid            |         1 |         543 | COMMIT /* xid=599 */                                                                          |
+---------------+-----+----------------+-----------+-------------+-----------------------------------------------------------------------------------------------+
6 rows in set (0.01 sec)
```

BEGIN和COMMIT对应，表示中间是一个事务。

use crashcourse由mysql添加，保证主备同步时更新到正确的库的表。Xid用于和redo log的日志关联。

```mysql
mysql> show warnings;
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                         |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1592 | Unsafe statement written to the binary log using statement format since BINLOG_FORMAT = STATEMENT. The statement is unsafe because it uses a LIMIT clause. This is unsafe because the set of rows included cannot be predicted. |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

```

因为当前binlog是statement格式且语句有limit所以命令可能是unsafe的。delete带limit很可能出现主备数据不一致。

1. 如果delete用索引a，那么会根据索引a找到a=4这一行
2. 如果用索引t_modified，删除的就是a=5这一行

因此可能出现主库用索引a，备库用t_modified。

#### row格式

重新插入a=4的行，`flush logs` 创建新的binlog文件方便查看，重新执行`delete` ，修改格式不会对已经写好的binlog生效。

```mysql
mysql> show binlog events in 'binlog.000008';
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
| Log_name      | Pos | Event_type     | Server_id | End_log_pos | Info                                 |
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
| binlog.000008 |   4 | Format_desc    |         1 |         125 | Server ver: 8.0.25, Binlog ver: 4    |
| binlog.000008 | 125 | Previous_gtids |         1 |         156 |                                      |
| binlog.000008 | 156 | Anonymous_Gtid |         1 |         235 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS' |
| binlog.000008 | 235 | Query          |         1 |         325 | BEGIN                                |
| binlog.000008 | 325 | Table_map      |         1 |         382 | table_id: 103 (crashcourse.t)        |
| binlog.000008 | 382 | Delete_rows    |         1 |         430 | table_id: 103 flags: STMT_END_F      |
| binlog.000008 | 430 | Xid            |         1 |         461 | COMMIT /* xid=613 */                 |
+---------------+-----+----------------+-----------+-------------+--------------------------------------+
7 rows in set (0.00 sec)
```

Table_map event表示要操作的表，Delete_rows event定义删除行为。从Pos看到，这个事务的binlog是从偏移量156开始。

`mysqlbinlog -vv /pathtomysql/data/binlog.000008 --start-position=156`

```sh
BEGIN
/*!*/;
# at 325
#210916 21:14:12 server id 1  end_log_pos 382 CRC32 0x245b85b3 	Table_map: `crashcourse`.`t` mapped to number 103
# at 382
#210916 21:14:12 server id 1  end_log_pos 430 CRC32 0xd6b18145 	Delete_rows: table id 103 flags: STMT_END_F

BINLOG '
JENDYRMBAAAAOQAAAH4BAAAAAGcAAAAAAAEAC2NyYXNoY291cnNlAAF0AAMDAxEBAAIBAQCzhVsk
JENDYSABAAAAMAAAAK4BAAAAAGcAAAAAAAEAAgAD/wAEAAAABAAAAFvlrwBFgbHW
'/*!*/;
### DELETE FROM `crashcourse`.`t`
### WHERE
###   @1=4 /* INT meta=0 nullable=0 is_null=0 */
###   @2=4 /* INT meta=0 nullable=1 is_null=0 */
###   @3=1541779200 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
# at 430
#210916 21:14:12 server id 1  end_log_pos 461 CRC32 0x4f04bd02 	Xid = 613
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;

```

- Table_map event跟`show binlog events` 看到的一样。如果操作多个表，每个表有一个对应的Table_map event，都会map到一个单独数字，区分对不同表的操作。

- -vv参数把内容都解析，从结果里可以看到各个字段的值，@1=4,@2=4
- binlog_row_image默认为FULL，因此Delete_event里包含了删掉的行的所有字段的值。如果是MINIMAL则只记录必要信息，在这个例子里，是id=4

当row格式时，binlog里记录了真实删除行的主键id，传到备库执行时肯定会删除id=4的行。

#### mixed格式

- statement格式可能导致主备不一致，所以要用row
- row很占空间。delete 十万行，statement只记录一句sql，占几十字节；row就要把这十万条记录写到binlog中。占空间多，耗IO资源，影响执行速度
- 所以有了折中的mixed，mysql自己判断sql是否可能引起主备不一致，有可能就用row，否则statement

#### 恢复数据

越来越多场景要求binlog为row格式，其中一个好处是恢复数据。

如果删错了数据，直接把binlog里的delete转成insert；如果执行错了insert，直接转成delete；对update，binlog会记录修改前整行和修改后整行，所以误执行update，只需把这个event前后的两行信息对调再去数据库里执行。

用binlog恢复数据的标准做法是，用mysqlbinlog工具解析，再把结果整个发给mysql执行，类似：

`mysqlbinlog master.000001 --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;`

将master.000001文件的第2738字节到第2973字节的内容解析出来放到mysql执行。

### 循环复制问题

![mysql基本主备切换流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/24/1.png)

上图是M-S结构，实际生产中使用比较多的是双M结构，即下图主备切换流程

![mysql基本主备切换流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/24/3.png)

AB总是互为主备关系，切换的时候不用修改主备关系。

建议参数`log_slave_updates` 设on，表示备库执行relay log后生成binlog。

循环复制指业务逻辑在A更新一条语句，把生成的binlog发给B，B执行完这条更新后也生成binlog，A也是B的备库，相当于又把B生成的binlog执行一遍，然后节点AB间不断循环执行这个更新语句。

从mysqlbinlog解析的结果可以看到binlog中记录了这个命令第一次执行时所在实例的server id。因此可以用以下逻辑解决循环复制：

1. 规定主备的server id必须不同
2. 备库重放binlog的过程中生成与原binlog的server id相同的binlog
3. 备库收到主库发来的binlog，先判断server id，如果与自己相同，丢弃日志

双M结构日志执行流：

1. A更新的事务，binlog里记的A的server id
2. 传到B执行一次，B生成的binlog记的也是A的server id
3. 日志传回给A，A判断server id与自己的相同，丢弃日志

## 25 mysql怎么保证高可用

### 主备延迟

数据同步有关的时间点：

1. 主库A执行完一个事务，写入binlog，记做T1
2. 备库B接收完这个binlog的时刻记做T2
3. 备库B执行完这个事务的时刻记做T3

主备延迟是同一个事务在备库执行完成的时间和主库执行完成的时间之差，即T3-T1

备库执行`show slave status` 会显示`seconds_behind_master` 表示备库延迟多少秒。

网络正常时，主备延迟的主要来源是T3-T2。最直接的表现是，备库消费relay log的速度比主库生产binlog的速度慢。

#### 主备延迟的来源

第一种可能，备库所在机器性能比主库所在机器性能差。

第二种可能，备库的压力大。一般备库提供读能力和运营需要的分析语句。主库直接影响业务，使用比较克制，反而忽略了备库的压力控制。一般可以选择

1. 一主多从。除了备库外，多接几个从库。（本专栏里主备切换中会被选成新主库的称备库，其他称从库）
2. 通过binlog输出到如hadoop的外部系统，让外部系统提供统计类查询的能力

第三种可能，大事务。比如一个事务在主库上执行10分钟，那么T3-T1就很可能为10分钟，导致从库延迟10分钟。

不要一次性用delete删除太多数据。这是典型的大事务场景。

另一种典型大事务，大表DDL。计划内的DDL，建议用gh-ost方案（github的开源方案）。

还有别的可能，作者没有展开。

### 主备切换的不同策略

#### 可靠性优先策略

![mysql可靠性优先主备切换流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/25/1.png)

在双M结构下从状态1到状态2切换的详细过程：

1. 判断备库B的`seconds_behind_master` ，如果小于某个值（如5s）继续下一步，否则持续重试这一步
2. 主库A的readonly设为true
3. 判断备库B的SBM，直到变为0
4. 备库B的readonly设为false
5. 业务请求切换到备库B

一般由专门的HA（highly available）系统完成。称为可靠性优先流程。

系统从步骤2开始不可用，到步骤5完成才恢复，其中步骤3占了大部分不可用时间。

#### 可用性优先策略

如果把上述步骤4、5调整到最开始执行，即不等主备数据同步，系统就几乎没有不可用时间。暂时称为可用性优先流程，代价是可能出现数据不一致。

例如表t有自增主键id，int型字段c。假设主库上其他表有大量更新，导致主备延迟5秒。在插入c=4后发起主备切换。假设binlog_format=mixed。

`insert into t(c) values(4);`

`insert into t(c) values(5);`

![mysql可用性优先且binlog为mixed](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/25/2.png)

1. A插入(4,4)，之后进行主备切换
2. 由于有5秒延迟，B没来得及应用插入c=4的relay log，就开始接收客户端插入c=5的命令
3. B插入(4,5)，并把这个binlog发给A
4. step5中B执行插入c=4的relay log，插入(5,4)。而在B执行的插入c=5，传到A就插入了(5,5)

最后有两行数据不一致。

row格式在记录binlog会记录新插入的行的所有字段值(实验即使binlog_row_image=MINIMAL也会)，所以最后只会有一行不一致，两边的主备同步线程会报错duplicate key error并停止。下图是可用性优先策略且binlog_format=row。

![mysql可用性优先且binlog为row](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/25/3.png)

因此row格式的binlog会让数据不一致问题更容易被发现。

一般来说，数据的可靠性的优先级高于可用性。

假设使用可靠性优先策略，`seconds_behind_master=1800` ，此时主库A断电，系统可能处于完全不可用的状态（不可读不可写），因为A掉电后可能不能让客户端连接直接切到备库B。如果直接切到备库B，保持B只读，relay log没有执行完成，客户端看不到在A已经执行完成的事务，会认为数据丢失。对一些业务来说，查询到“暂时丢失数据的状态”也是不能接受的。

在满足可靠性前提下，mysql的高可用性，是依赖于主备延迟的。延迟时间越小，主库故障时服务恢复需要时间就越短，可用性越高。

## 26 备库为什么会延迟几个小时

### 主备同步sql_thread多线程模型

当备库执行relay log的速度赶不上主库产生binlog的速度，主备延迟会越来越严重。

![mysql主备同步流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/24/2.png)

在主备同步流程中，关注start$$\rightarrow$$ undolog(mem)和sql_thread$$\rightarrow$$ DATA两个箭头。前者代表客户端写入主库，后者代表备库上sql_thread执行relay log。前者的并行度远高于后者。

InnoDB支持行锁，因此对并发度的支持很友好。而日志在备库执行如果用单线程就会造成主备延迟。

在mysql5.6前，mysql只支持单线程复制。

多线程复制机制是把只有一个线程的sql_thread改成符合如下模型

![sql_thread多线程模型](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/26/1.png)

coordinator就是原来的sql_thread，在这个模型中只负责读取中转日志relay log和分发事务。执行更新数据的变为worker线程。worker的个数由参数`slave_parallel_workers` 决定。

coordinator在分发的时候满足两个要求：

1. 更新同一行的两个事务，必须分发到同一个worker中
2. 同一个事务不能拆开，必须放到同一个worker中

### mysql5.5的并行复制策略（作者自己写的）

官方mysql5.5只有单线程复制。

#### 按表分发策略

基本思路是两个事务更新不同的表，就可以并行。下图为按表并行复制线程模型

![按表并行复制线程模型](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/26/2.png)

每个worker对应一个hash表，保存当前worker“执行队列”里的事务涉及的表。hash表的key是“库名.表名”，value是数字，表示队列中有多少事务修改这个表。

有事务分配给worker时，事务涉及的表被添加到对应hash表中，worker执行完成后这个表从hash表中去掉。

假设coordinator从relay log读入新事务T，T修改的行涉及表t1和t3

1. 由于T涉及修改t1，worker_1队列中有事务要修改t1，T和队列中某个事务要修改同一个表，称T和worker_1是冲突的。
2. 按1的逻辑，顺序判断T和每个worker的冲突关系。发现T和worker_2也冲突
3. T跟多于1个worker冲突，coordinator进入等待
4. 每个worker继续执行并修改hash表。假设`hash_table_2` 里涉及到修改t3的事务先执行完成，`hash_table_2` 中`db1.t3` 这一项被去掉。
5. coordinator发现跟T冲突的只有worker_1，把T分配给worker_1
6. coordinator继续读下一个relay log，继续分配事务。

每个事务被分发时，跟所有worker冲突关系有3种：

1. 跟所有worker不冲突，被coordinator线程分配给最空闲的worker
2. 跟多于1个worker冲突，coordinator线程进入等待，直到存在冲突关系的worker只有1个
3. 只跟1个worker冲突，coordinator线程把这个事务分配给冲突的worker

这个方案在多个表负载均衡的场景效果很好。但是遇上热点表，即如果所有更新事务都会涉及某一个表，所有事务都被分配到同一个worker，就变成单线程了。

#### 按行分发策略

如果两个事务没有更新相同的行，就可以在备库上并行执行。要求binlog格式为row（日志中记录必要的字段值以区分行）。

hash表的key变为“库名+表名+唯一键的值”，唯一键只有主键还不够。

```mysql
CREATE TABLE `t1` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `a` (`a`)
) ENGINE=InnoDB;
 
insert into t1 values(1,1,1),(2,2,2),(3,3,3),(4,4,4),(5,5,5);
```



|            sessionA             |            sessionB             |
| :-----------------------------: | :-----------------------------: |
| `update t1 set a=6 where id=1;` |                                 |
|                                 | `update t1 set a=1 where id=2;` |

两个事务要更新的行主键不同，如果被分到不同worker，B先执行，id=1的行中a还是1，会报唯一键冲突。

因此基于行的策略，hash表中还需要考虑唯一键，key应该是“库名+表名+索引a的名字+a的值”

比如B的语句执行后，binlog里会记录修改前和修改后整行数据的值。coordinator解析这个binlog时，hash表有三项：

1. key=hash_func(db1+t1+"PRIMARY"+2),value=2;value=2是因为修改前后id的值不变，出现了2次（前文作者说value表示有多少事务修改当前表\行，这里的语义应该变为mysqlbinlog解析出来的整行数据的值出现的次数）
2. key=hash_func(db1+t1+"a"+2),value=1，表示会影响到这个表a=2的行
3. key=hash_func(db1+t1+"a"+1),value=1，表示会影响到这个表a=1的行

相比按表并行分发策略，这个策略在决定线程分发时，消耗更多计算资源。两个方案都有一些约束条件（其实按表的方案没那么严格，作者这里一起说了）：

1. 要从binlog解析出表名、主键值和唯一索引的值，主库的binlog必须为row
2. 表必须有主键（按照之前搜索的，如果没有主键，会把唯一键当主键处理，上述例子中没有主键似乎也可以）
3. 表不能有外键。如果有外键，级联更新的行不会记录在binlog，冲突检测不准确。

按行分发的策略显然并行度更高。如果是操作很多行的大事务，按行分发的策略有两个问题：

1. 耗费内存。比如一条语句要删100万行，hash表就要记录100万个项（按上文，应该是至少）
2. 耗费cpu。解析binlog，然后计算hash值，对于大事务成本很高。

所以作者在实现按行分发策略时设置一个阈值，单个事务修改的行如果超过设置行数阈值，就暂时退化为单线程模型。退化逻辑：

1. coordinator先不分发这个事务
2. 等所有worker执行完成
3. coordinator直接执行这个事务
4. 恢复并行模型

### mysql5.6的并行复制策略

支持的并行复制粒度是按库并行。hash表里key是库名。

如果主库上有多个库，并且各个库压力均衡，效果就很好。

比起作者写的策略，有两个优势。一是hash快，只需要库名；且因为一个实例上库的数不会很多，hash表的项不会很多。二是不要求binlog格式，statement格式也很容易拿到库名。

### MariaDB的并行复制策略

利用redo log组提交的特性：

1. 同一组提交的事务一定不会修改同一行（事务提交之后行锁才释放）
2. 主库上可以并行执行的事务，备库上也一定可以

MariaDB的实现：

1. 在一组里一起提交的事务，有一个相同的commit_id，下一组就是commit_id+1
2. commit_id直接写到binlog
3. 传到备库应用时，相同commit_id的事务分发到多个worker
4. 这一组全部执行完成，coordinator再取下一组

没有完全实现模拟主库的并发度，由于步骤4，达不到主库的流水线效果。主库在committing一组事务时，下一组事务在running，而备库不行。

容易被大事务拖后腿，同一组内如果有超大事务，就必须等待执行完成才能开始下一组。

### mysql5.7的并行复制策略

mysql5.7由参数`slave-parallel-type` 控制并行复制策略：

1. 配置为DATABASE，表示使用5.6的按库并行
2. 配置为LOGICAL_CLOCK，表示类似MariaDB的策略，但是对并行度做了优化。

同时处于“执行状态”的事务不能并行，因为可能有由于锁冲突而处于等待的事务，如果这些事务在备库上分配到不同worker，可能造成主备数据不一致。

MariaDB策略的核心是所有“处于commit”状态的事务可以并行，已经通过了锁冲突的检验。

![两阶段提交细化](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/23/3.png)

实际上只要到达redo log prepare阶段就已经通过锁冲突检验（事务逻辑执行完成，内存已经写好，要写到page cache的阶段）。

因此mysql5.7并行复制策略的思想是：

1. 同时处于prepare的事务在备库可以并行
2. 同一时刻处于prepare的事务和处于commit的事务，在备库也可以并行

binlog组提交相关的参数`binlog_group_commit_sync_delay` 和`binlog_group_commit_sync_no_delay_count` 在5.7的并行复制策略里，可以用来制造更多“同时处于prepare的事务”，增加备库复制的并行度。

这两个参数可以达到故意让主库提交慢些，以及让备库执行快些的效果。在5.7处理备库延迟时，可以考虑调整这两个参数达到提升备库并行复制并发度的目的。

### mysql5.7.22的并行复制策略

新增基于WRITESET的并行复制。参数`binlog-transaction-dependency-tracking` 控制是否启动这个策略：

1. 设为COMMIT_ORDER，表示上文根据同时进入prepare和commit来判断是否可以并行的策略
2. WRITESET，表示对事务涉及更新的每一行，计算这一行的hash值，组成集合writeset。如果两个事务的writeset没有交集就可以并行。
3. WRITE_SESSION，在WRITESET上增加约束，在主库上同一个线程先后执行的两个事务，在备库执行时保证相同的先后顺序。

hash值通过“库名+表名+索引名+索引值”计算，如果表上除了主键索引，还有唯一索引，对每个唯一索引，insert语句对应的writeset就要多一个hash值。

跟作者自己写的5.5版本的按行分发的策略相比，有以下优势：

1. writeset在主库生成后直接写到binlog，备库执行时就不需要解析binlog event里的行数据，节省计算量
2. 不需要把整个事务的binlog扫完才决定分发的worker，节省内存
3. 备库分发不依赖于binlog内容，statement格式也可以。

对于“表上没主键”和“有外键约束”的场景，WRITESET策略也会暂时退化为单线程模型。

注意这里也说明binlog协议并不是向更早兼容的，所以主备切换、版本升级也要考虑这一点。

## 27 主库出问题，从库怎么处理(一主多从的切换正确性)

一主多从的设置，一般用于读写分离，主库负责所有写入和一部分读，其他读请求由从库分担。

一主多从基本结构和主备切换后的结果：

![一主多从基本结构](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/27/1.png)

![一主多从基本结构主备切换](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/27/2.png)



### 基于位点的主备切换

当把B设置成A'的从库时需要执行change master命令：

```mysql
CHANGE MASTER TO
MASTER_HOST=$host_name
MASTER_PORT=$port
MASTER_USER=$user_name
MASTER_PASSWORD=$password
MASTER_LOG_FILE=$master_log_name
MASTER_LOG_POS=$master_log_pos
```

主库对应的文件名和日志偏移量就是同步位点。

B原本记录的是A的位点，相同的日志，A和A'的位点是不同的。从库B切换的时候要先找同步位点。

为了防止丢数据，往往要把同步的位点“提前”一些，重复执行一些事务。

一种取同步位点的方法：

1. 等待新主库A'把relay log全部同步完成。
2. 在A'上执行`show master status` ，得到A'上最新的binlog文件名和写入位置
3. 取原主库A故障的时刻T
4. 用mysqlbinlog解析A'的binlog文件，得到T时刻的位点

`mysqlbinlog File -stop-datetime=T -startdatetime=T`

得到的`end_log_pos` （假设是123）的值就是A'实例在T时刻写入的新的binlog的位置，把这个值作为`$master_log_pos` 用在B的`change master` 命令里

这个值不精确。

假设T时刻，A执行完成一条`insert` 插入了一行R，且binlog已经传给A'和B，在传完的瞬间A掉电。

1. 在B上，由于同步了binlog，R已经存在
2. 在A'上，R也已经存在，日志写在123这个位置之后
3. 在B上执行`change master` ，执行A'的123位置，会把插入R这一行的binlog又同步到B执行。

这时候B的同步线程会报`Duplicate entry 'id_of_R' for key 'PRIMARY'` 提示主键冲突，停止同步。

 所以通常在切换时要先主动跳过这些错误，有两种常用方法。

一是主动跳过一个事务。

```mysql
set global sql_slave_skip_counter=1;
start slave;
```

`change master` 过程中可能不止重复执行一个事务，需要我们在B刚开始接到新主库A'时持续观察，每次碰到错误就停下执行一次跳过命令。

另一种是设置`slave_skip_errors` 参数，直接设置跳过指定错误。

主备切换时，`change master` 再同步日志常遇到两类错误

- 1062错误，插入数据时唯一键冲突
- 1032错误，删除数据时找不到行。

可以把`slave_skip_errors` 设置为"1032,1062"，中间碰到这两个错误直接跳过。

设置的背景是，在主备切换过程中，直接跳过这两类错误是无损的。等主备间的同步关系建立完成并稳定执行一段时间后，需要重新把这个参数设置为空，以免之后真的出现主从数据不一致也跳过。

### GTID

`sql_slave_skip_counter` 和`slave_skip_errors` 的方法操作复杂容易出错。mysql5.6引入GTID。

GTID（Global Transaction Identifier，全局事务ID），是一个事务在提交时生成的唯一标识，格式是

`GTID=server_uuid:gno`

- `server_uuid` 是一个mysql实例第一次启动时自动生成的，是一个全局唯一值
- gno是一个整数，初始值为1，每次提交事务时分配给事务并加1

mysql官方文档里GTID的定义：

`GTID=source_id:transaction_id`

其实跟作者定义的一样。mysql里说`transaction_id`一般指事务id（第八章），在事务执行过程中分配，如果事务回滚也会递增。作者为避免混淆改成gno，gno在事务提交时才分配。

启动GTID模式，在启动mysql实例时，加上参数`gtid_mode=on` 和`enforce_gtid_consistency=on`

GTID模式下事务跟GTID一一对应。session变量`gtid_next`  的值决定GTID的生成：

1. `gtid_next=automatic` ，mysql会把`server_uuid:gno` 分配给事务。记录binlog 时，先写一行`SET @@SESSION.GTID_NEXT='server_uuid:gno'` ，并把这个GTID加入本实例的GTID集合
2. `gtid_next` 是指定值，如`set gtid_next='current_gtid'` ，有两种可能。current_gtid已经在本实例的gtid集合中，接下来执行的这个事务会被系统忽略；否则这个current_gtid分配给接下来要执行的事务，系统不需要给这个事务生成GTID，gno不用加1。这个事务提交后如果要执行下一个事务，需要重新设置`gtid_next`

假设从库X要同步一个会主键冲突的事务，可以执行

```mysql
set gtid_next='引起冲突事务的gtid';
begin;
commit;
set gtid_next=automatic;
start slave;
```

通过提交空事务把GTID加入实例的GTID集合，通过`show master status` 可以看到集合里的GTID

#### 基于GTID的主备切换

在GTID模式，从库B要设置为新主库A'的从库的语法为

```mysql
CHANGE MASTER TO
MASTER_HOST=$host_name
MASTER_PORT=$port
MASTER_USER=$user_name
MASTER_PASSWORD=$password
master_auto_position=1
```

`master_auto_position=1` 表示这个主备关系用GTID协议。

假设现在这个时刻A'的GTID集合为set_a，B的为set_b。在实例B上执行`start slave` ，取binlog的逻辑是：

1. B指定主库A'，基于主备协议建立连接
2. B把set_b发给A'
3. A'算出set_a与set_b的差集，判断A'本地是否包含这个差集需要的所有binlog事务。
   1. 不全部包含，表示A'已经把B需要的binlog删掉了，直接返回错误
   2. 全部包含，A'从自己的binlog文件里找出第一个差集中的事务，发给B

4. 从这个事务开始往后读文件，按顺序取binlog发给B执行。

引入GTID后，从库B、C、D分别执行`change master` 指向实例A'即可。

在GTID模式下，找位点的工作在A'内部自动完成，日志的完整性由A'判断；在基于位点的主备切换中，日志位置由从库指定，主库不做完整性判断。

GTID也可以解决前文提到过的循环复制问题。

### GTID和在线DDL

第22章提过如果业务高峰的慢查询是由于索引缺失引起的，可以通过在线加索引解决。为避免新增索引对主库性能造成影响，一般先在备库加，再切换。

一般来说，通过主备切换来实现在线DDL的操作，这里的DDL指增删索引、删最后一列、加最后一列。

在双M结构，备库的DDL语句也会传给主库，影响主库性能，可以通过GTID解决。

假设主库X，备库Y，都打开GTID模式。

1. X上执行`stop slave`
2. Y上执行DDL，不需要关闭binlog
3. 查出DDL对应的GTID，记做`server_uuid_of_Y:gno`
4. X上执行以下语句

```mysql
set GTID_NEXT="server_uuid_of_Y:gno";
begin;
commit;
set GTID_NEXT=automatic;
start slave;
```

既可以让Y有DDL的binlog，也确保不会在X上执行。完成主备切换后，按上述流程再执行一遍让X上也有对应的DDL（如增加索引）即可。

### GTID模式下新的从库接上主库，需要的binlog已经没了怎么做

> 1. 如果业务允许主从不一致的情况，那么可以在主库上先执行 `show global variableslike  ‘gtid_purged’;` ，得到主库已经删除的 GTID 集合，假设是 `gtid_purged1` ；然后先在从库上执行 `reset  master` ，再执行 `set global gtid_purged=‘gtid_purged1’;` ；最后执行 `start  slave` ，就会从主库现存的 binlog 开始同步。binlog 缺失的那一部分，数据在从库上就可能会有丢失，造成主从不一致。
> 2. 如果需要主从数据一致的话，最好还是通过重新搭建从库来做。
> 3. 如果有其他的从库保留有全量的 binlog 的话，可以把新的从库先接到这个保留了全量binlog 的从库，追上日志以后，如果有需要，再接回主库
> 4. 如果 binlog 有备份的情况，可以先在从库上应用缺失的 binlog，然后再执行 `start slave` (这里我的理解是回到第1步的情况)。

## 28 读写分离的坑

![读写分离基本结构](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/27/1.png)

client主动做负载均衡，数据库连接信息放在客户端连接层。

另一种架构师mysql和客户端之间有一个中间代理层proxy，客户端连接proxy，由proxy分发路由。

直连方案少了转发查询性能稍好。架构简单、排查问题方便，但是主备切换、库迁移客户端都会感知到并需要调整。一般这样的架构伴随负责管理后段的组件比如Zookeeper。

proxy架构上述工作由proxy完成，整体复杂。目前趋势是往带proxy的架构发展。

由于主从延迟，客户端执行完更新事务马上发起查询，选择从库就又可能读到事务更新之前的状态。

在从库上读到系统的一个过期状态的现象，在这里暂且称为“过期读”。

### 强制走主库方案

通常把查询分两类:

1. 必须拿到最新结果的请求，强制发到主库。比如交易平台卖家发布商品查看是否发布成功。
2. 可以读到旧数据的请求才发到从库。交易平台买家逛商铺，晚几秒看到最新商品也可以接受。

这个方案用得最多。

金融类业务所有查询都不能过期读，读写压力都在主库。下文主要讨论可以读写分离的场景解决过期读的方案。

### sleep方案

方案的假设是大多数情况主备延迟在1秒以内。

主库更新后，读从库之前sleep以下，类似于执行`select sleep(1)`

交易平台卖家的场景，可以通过AJAX避免查数据库，先显示商品。

存在不精确的问题，本来0.5秒可以从从库读到正确结果的请求也会等1秒，主从延迟超过1秒还是会过期读。

### 判断主备无延迟方案

三种确保主备无延迟的方法：

1. 从库执行查询请求前，判断`seconds_behind_master` 是否为0，否则等到变0才执行查询。在并行复制的情况下，SBM的值非常不精确。
2. 对比位点确保主备无延迟。
   - `show slave status` 的部分结果![showslavestatus结果](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/28/1.png)
   - `Master_Log_File` 和`Read_Master_Log_Pos` 表示读到的主库的最新位点
   - `Relay_Master_Log_File` 和`Exec_Master_Log_Pos` 表示备库执行的最新位点
   - 两组值完全相同表示接收到的日志已经同步完成

3. 对比GTID集合确保主备无延迟
   - Auto_Position=1表示这对主备关系用GTID协议
   - Retrieve_Gtid_Set表示备库收到的所有日志的GTID集合
   - Executed_Gtid_Set表示备库所有已经执行完成的GTID集合。两个集合相同表示备库接收到的日志同步完成。

仍然可能不精确。可能有一部分事务主库已经提交，binlog已经持久化，并反馈给客户端，客户端认为数据已经更新完成，但是对应事务的binlog备库还没有收到，并且这时从库收到的日志都已经同步完成。

此时从库认为没有延迟，但查询请求到从库仍然会过期读。

### 配合semi-sync

半同步复制，semi-sync replication

semi-sync的设计：

1. 事务提交时主库把binlog发给从库
2. 从库收到binlog，回主库一个ack
3. 主库收到ack，才能回复客户端“事务完成”

启动semi-sync，确保所有给客户端发送过确认的事务，备库都已经收到对应binlog日志。

semi-sync配合位点和GTID判断的方案可以确定在从库上执行的查询请求避免过期读。但是只对一主一从场景成立。

一主多从下，主库收到一个从库的ack就给客户端返回确认。这时在从库执行查询请求，如果查询落在回复主库ack的从库上，可以确保读到最新数据；否则仍然可能过期读。

另一个潜在问题是，业务更新高峰，主库位点或者GTID集合更新很快，有可能判断无延迟的逻辑一直不成立，从库上一直无法响应查询请求。对应下面时序图。

![主备持续延迟一个事务](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/28/2.png)

主备一直延迟一个事务，使用判断无延迟的方案，select直到状态4都无法执行，但实际上在状态3执行查询已经可以得到write trx1对应的预期结果。

### 等主库位点方案

`select master_pos_wait(file, pos[, timeout]);`

- 这条命令在从库执行
- file和pos指主库上的文件名和位置(`show master status` )
- timeout可选，正整数N表示函数最多等待N秒

返回值：

- 正整数M，表示从命令执行开始，到应用完file和pos表示的binlog位置，执行了多少事务。
- NULL，表示执行期间备库同步线程发生异常
- -1，表示timeout超时
- 0，表示刚开始执行的时候就已经执行过file和pos指定位置了。

对于上节先执行trx1，再执行查询请求的逻辑，可以保证查到正确数据：

1. trx1事务更新完成后，马上执行`show master status` 得到当前主库binlog对应的File和Pos
2. 选定一个从库执行查询
3. 在从库上执行`select master_pos_wait(File,Pos,1);` （假设最多等待1秒）
4. 返回值如果为>=0的正整数则在这个从库执行查询
5. 否则到主库执行查询

### GTID方案

`select wait_for_executed_gtid_set(gtid_set[, timeout]);`

1. 在从库执行
2. 等待直到这个库执行过的事务中包含传入的gtid_set，返回0
3. 超时返回1

从mysql5.7.6开始，允许在执行完更新类事务后把事务的GTID返回给客户端，这个方案可以减少一次查询。还是以时序图的例子举例执行流程：

1. trx1执行完成后，从返回包里获取这个事务的GTID，记做gtid1
2. 选定一个从库执行查询
3. 从库上执行`select wait_for_executed_gtid_set(gtid1, 1);` （假设最多等待1秒）
4. 返回值为0则在这个从库查询
5. 否则到主库查询

将参数`session_track_gtids` 设为`OWN_GTID` ，可以让mysql执行事务后在返回包中带GTID，然后通过API接口`mysql_session_track_get_first` 从返回包中解析出GTID（或者编程语言类似的函数）。

### 小结

实际应用中几个方案混合使用。

比如，客户端对请求做分类，判断哪些请求可以接受过期读，对不能接受过期读的请求，再使用等GTID或者等主库位点的方案。

## 29 如何判断数据库是否出问题

### select1判断不准

```mysql
set global innodb_thread_concurrency=3;
```

|          sessionA           |              B              |              C              |                         D                          |
| :-------------------------: | :-------------------------: | :-------------------------: | :------------------------------------------------: |
| `select sleep(100) from t;` | `select sleep(100) from t;` | `select sleep(100) from t;` |                                                    |
|                             |                             |                             | `select 1;` (OK)<br /> `select * from t;`(blocked) |

`innodb_thread_concurrency` 控制InnoDB并发线程（并发查询）上限，0为不限制。一旦并发线程达到这个值，InnoDB接收到新请求的时候就进入等待，直到有线程退出。

通常把它设置为[64,128]。

并发连接和并发查询不同。`show processlist` 的结果看到的是并发连接，当前正在执行的语句才是并发查询。并发连接多只是多占内存而已，并发查询才占cpu。

线程进入锁等待后，并发线程计数减1。因为锁等待的线程不吃CPU。必须这样设计才能避免整个系统锁死。

### 查表判断仍不准

为检测InnoDB并发线程过多导致不可用，需要构造一个访问InnoDB的场景。

一般在系统库（mysql库）里创建一个表，只放一行数据，定期执行：

`select * from mysql.health_check;`

binlog所在磁盘空间占用达到100%后，所有更新语句和commit语句都会阻塞，但是还可以正常读数据。

### 更新判断

常见做法是在检测表里放一个timestamp字段。

`update mysql.health_check set t_modified=now();`

可用性检测要包含主库和备库。一般把主（A）备（B）关系设置为双M结构，备库上执行上述命令也要发回给A，所以，A的同步线程和检测语句可能出现行冲突(主键冲突，同时执行的话，都是insert行为，在binlog里记write rows event)，导致主备同步停止。

为了主备之间的更新判断不产生冲突，在检测表里存入多行数据，用A、B的server_id做主键。

```mysql
CREATE TABLE `health_check` (
  `id` int(11) NOT NULL,
  `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB;
 
/* 检测命令 */
insert into mysql.health_check(id, t_modified) values (@@server_id, now()) on duplicate key update t_modified=now();
```

#### 判定慢

所有的检测逻辑都有一个超时时间N，执行update超过N秒不返回，就认为系统不可用。

假设一个日志盘的IO利用率是100%，这时候系统响应非常慢，已经需要主备切换。

但是系统IO是正常工作的，每个请求都有机会拿到IO，检测使用的update需要的资源很少，可能在拿到IO的时候就提交成功，在超时之前就返回给检测系统，于是得到“系统正常”的结论。

以上都是外部检测，具有随机性。系统已经出问题，需要等下一轮检测轮询的时候才可能发现问题，如果发现不了，会导致主备不及时切换。

### 内部统计

mysql5.6后提供的`performance_schema` 库在`file_summary_by_event_name` 表统计每次IO请求的时间。

```mysql
mysql> select * from performance_schema.file_summary_by_event_name where event_name='wait/io/file/innodb/innodb_log_file'\G
*************************** 1. row ***************************
               EVENT_NAME: wait/io/file/innodb/innodb_log_file
               COUNT_STAR: 2496
           SUM_TIMER_WAIT: 630086418000
           MIN_TIMER_WAIT: 584000
           AVG_TIMER_WAIT: 252438000
           MAX_TIMER_WAIT: 23573375000
               COUNT_READ: 8
           SUM_TIMER_READ: 483251000
           MIN_TIMER_READ: 667000
           AVG_TIMER_READ: 60406000
           MAX_TIMER_READ: 271375000
 SUM_NUMBER_OF_BYTES_READ: 70656
              COUNT_WRITE: 1320
          SUM_TIMER_WRITE: 197672582000
          MIN_TIMER_WRITE: 2042000
          AVG_TIMER_WRITE: 149751000
          MAX_TIMER_WRITE: 3263667000
SUM_NUMBER_OF_BYTES_WRITE: 1098240
               COUNT_MISC: 1168
           SUM_TIMER_MISC: 431930585000
           MIN_TIMER_MISC: 584000
           AVG_TIMER_MISC: 369803000
           MAX_TIMER_MISC: 23573375000
1 row in set (0.01 sec)

```

这一行表示redo log的写入时间，第一列表示统计类型。

第一组五列，是所有IO类型的统计。`COUNT_STAR` 是所有IO的总次数，接着4项是具体统计项，单位皮秒。

第二组六列，读操作的统计。`SUM_NUMBER_OF_BYTES_READ` 统计总共从redo log里读了多少字节。

第三组六列，统计写操作。

第四组是对其他操作类型的统计，在redo log里，可以认为是对`fsync` 的统计。

上述是redo log，binlog对应`event_name="wait/io/file/sql/binlog"` 这一行。

打开这个统计功能是有损性能的。

用下列语句打开或关闭某个统计项

`update setup_instruments set ENABLED='YES', Timed='YES' where name like '%wait/io/file/innodb/innodb_log_file%';`

假设IO请求超过200毫秒属于异常

`select event_name,MAX_TIMER_WAIT FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;`

查询结果不为空则有异常。取完需要的信息后，通过下列语句清空统计信息，以后再出现异常，就可以加入监控累积值。

`truncate table performance_schema.file_summary_by_event_name;`

## 30 答疑（二）：用动态的观点看加锁

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;
 
insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

### 怎么看死锁

锁是在执行过程中随着遍历或查找索引一个个加的，而不是一次性加上去的。

```mysql
begin;
select id from t where c in (5,20,10) lock in share mode;
```

查找c=5，加锁顺序是先锁住(0,5]，然后加间隙锁(5,10)。

然后查找c=10，加锁顺序是(5,10]，然后(10,15)。

最后查找c=20，加锁顺序是(15,20]，然后(20,25)

同时有另一个语句:

`select id from t where c in (5,20,10) order by c desc for update;`

加锁范围一样，但是`order by c desc` ，使得加锁顺序是先锁c=20,再c=10，最后c=5

当两条语句要加锁相同资源，但是加锁顺序相反，并发时可能出现死锁。

自己的实验模拟：

对话A

```mysql
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> select sleep(5), id from t where c in(5,20,10) lock in share mode;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction

```

同时另一个对话B

```mysql
mysql> select sleep(5), id from t where c in(5,20,10) order by c desc for update
;
+----------+----+
| sleep(5) | id |
+----------+----+
|        0 | 20 |
|        0 | 10 |
|        0 |  5 |
+----------+----+
3 rows in set (19.64 sec)
```

出现死锁后，通过`show engine innodb status;` 中的`LATESTDETECTED DEADLOCK` 一节，得到死锁信息（已经解开死锁的也能看，mysql保留最后一个死锁的现场，但是这个现场不完备）。

```mysql
------------------------
LATEST DETECTED DEADLOCK
------------------------
2021-09-20 17:13:16 0x30e80b000
*** (1) TRANSACTION:
TRANSACTION 3415369, ACTIVE 10 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1136, 4 row lock(s)
MySQL thread id 68, OS thread handle 13137600512, query id 682 localhost root User sleep
select sleep(5), id from t where c in(5,20,10) order by c desc for update

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 49 page no 5 n bits 80 index c of table `crashcourse`.`t` trx id 3415369 lock_mode X
Record lock, heap no 6 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
 1: len 4; hex 80000014; asc     ;;


*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 49 page no 5 n bits 80 index c of table `crashcourse`.`t` trx id 3415369 lock_mode X waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 4; hex 8000000a; asc     ;;


*** (2) TRANSACTION:
TRANSACTION 421717344324936, ACTIVE 10 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1136, 5 row lock(s)
MySQL thread id 67, OS thread handle 13137903616, query id 681 localhost root User sleep
select sleep(5), id from t where c in(5,20,10) lock in share mode

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 49 page no 5 n bits 80 index c of table `crashcourse`.`t` trx id 421717344324936 lock mode S
Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000005; asc     ;;
 1: len 4; hex 80000005; asc     ;;

Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 4; hex 8000000a; asc     ;;


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 49 page no 5 n bits 80 index c of table `crashcourse`.`t` trx id 421717344324936 lock mode S waiting
Record lock, heap no 6 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000014; asc     ;;
 1: len 4; hex 80000014; asc     ;;

*** WE ROLL BACK TRANSACTION (2)

```

1. - (1)TRANSACTION是第一个事务的信息
   - (2)TRANSACTION同理
   - WE ROLLBACK TRANSACTION(2)表示最终回滚第二个事务。

2. 第一个事务中：
   - WAITING FOR THIS LOCK TO BE GRANTED表示在等待的锁信息
   - index c of table crashcourse.t 等的是表t的索引c上的锁
   - lock_mode X waiting表示这个语句要加写锁，等待中
   - Record lock说明是记录锁
   - n_fields 2 表示记录是两列，即字段c和主键id
   - 0: len 4; hex 8000000a; asc     ;; 是第一个字段，即c，值是16进制a，即10
   - 1: len 4; hex 8000000a; asc     ;; 是主键id，值是10
   - asc表示打印出值里的“可打印字符”，10不是可打印字符，显示空格
   - HOLDS THE LOCK(S)表示持有哪些锁
   - 两个hex 80000014表示持有(c=20,id=20)的记录锁

3. 第二个事务中：
   - 两个hex 80000005和两个hex 8000000a表示这个事务持有(c=5,id=5)和(c=10,id=10)这两个记录锁
   - WAITING FOR THIS LOCK TO BE GRANTED表示在等(c=20,id=20)的记录锁

所以`for update` 这条持有c=20的记录锁，在等c=10的锁。`lock in share mode` 持有c=5和c=10两个记录锁，在等c=20的锁。

因此导致死锁。因此得到锁是一个个加的。要避免死锁，对同一组资源，按照尽量相同的顺序访问。

在作者原文中，占有资源更多的语句回滚成本更大，所以选择了回滚成本更小的语句；但是我的实验结果是选择回滚占有锁资源更多的第二个事务，即`lock in share mode` ，猜测是因为加写锁\写语句（`for update` )的优先级更高。

### 怎么看锁等待

|                              A                               |                              B                               |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| `begin;` <br />`select * from t where id>10 and id<=15 for update;` |                                                              |
|                                                              | `delete from t where id=10;` (OK)<br />`insert into t values(10,10,10);` (blocked) |

通过`show engine innodb status\G` 查看结果的TRANSACTION这一节。(锁住的时候才能看到)

```mysql
---TRANSACTION 3415383, ACTIVE 3 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 68, OS thread handle 13137600512, query id 700 localhost root update
insert into t values(10,10,10)
------- TRX HAS BEEN WAITING 3 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 49 page no 4 n bits 80 index PRIMARY of table `crashcourse`.`t` trx id 3415383 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 4; hex 8000000f; asc     ;;
 1: len 6; hex 000000341d40; asc    4 @;;
 2: len 7; hex 82000000ac0137; asc       7;;
 3: len 4; hex 8000000f; asc     ;;
 4: len 4; hex 8000000f; asc     ;;

```

1. index PRIMARY of table crashcourse.t 表示这个语句被锁住是因为表t主键索引上的某个锁
2. lock_mode X locks gap before rec insert intention waiting 表示当前线程要加的是一个插入意向锁（可以认为是插入动作本身），gap before rec表示间隙锁
3. gap在哪个记录之前，0～4表示这个记录的信息
4. n_fields 5表示这个记录有5列：
   - 0: len 4; hex 8000000f; asc     ;;主键id字段，id=15，这个间隙是id=15之前的，id=10不存在了，所以间隙锁是(5,15)
   - 1: len 6; hex 000000341d40; asc    4 @;;长度为6字节的事务id，表示最后修改这一行的是trx id为多少的事务
   - 2: len 7; hex 82000000ac0137; asc       7;;长度为7字节的回滚段（undo log？）信息
   - 后两列是c和d的值，都是15

由于delete把id=10的行删掉，两个间隙(5,10),(10,15)变为一个(5,15)

## 31 误删数据怎么办？

### 误删行

可以用Flashback工具恢复。

原理是修改binlog内容并重放，要求`binlog_format=row` 和`binlog_row_image=FULL`

- 对于误insert，对应的binlog event是Write_rows event，把它改成Delete_rows event
- 对误delete，和insert相反
- 对误Update_rows event，binlog记录了修改前后的整行数据，对调位置即可。

如果误操作多行，恢复时逆序操作；涉及多个事务，恢复时逆序执行。

不建议在主库上执行。比较安全的做法，是恢复出一个备份，或者找一个从库作为临时库，在临时库上执行，再将确认过的临时库的数据，恢复回主库。主库上如果发现误操作的时间晚了，可能业务代码已经在错误数据的基础上修改了其他数据，这时如果单独恢复误操作，可能对数据二次破坏。

建议参数`sql_safe_updates` 设为on，效果是delete或者update语句没有where，或者where里没有包含索引字段，执行会报错。

在上述前提下，确定操作没问题，删除全表可以用where id>=0

考虑性能，比起delete全表，优先用`truncate table` 或`drop table`

`truncate/drop table` 和`drop database` 无法通过Flashback恢复，因为即使`binlog_format=row` ，这三个命令的binlog还是statement格式，只有语句没有数据。

### 误删库/表

需要全量备份加增量日志，要求线上有定期的全量备份并且实时备份binlog

#### mysqlbinlog方案

假设一个库一天一备，上次备份是当天0点，有人中午12点误删了一个库

1. 取最近一次全量备份
2. 用备份恢复出一个临时库
3. 从binlog备份里取出0点之后日志
4. 把这些日志除了误删除语句外全部应用到临时库

![mysqlbinlog方法恢复数据流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/31/1.png)

mysqlbinlog加上`-database` 参数指定误删表所在的库，避免在恢复数据时还要应用其他库的日志。

这个方法不够快，原因有两个：

1. 如果是误删表，最好是只重放这张表的操作，但是mysqlbinlog不能指定只解析一个表的日志。
2. mysqlbinlog解析出日志应用，应用日志的过程只能单线程。并行复制用不上。

#### master-slave方案

一种加速方法是，用全量备份恢复出临时实例后，将它设置成线上备库的从库，好处有

1. `start slave` 前先执行`change replication filter replicate_do_table=(tbl_name)` 可以让临时库只同步误操作的表
2. 可以用并行复制技术

过程示意图如下：

![master-slave方法恢复数据流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/31/2.png)

虚线代表如果因为时间太久，备库上已经删掉临时实例需要的binlog，可以从binlog备份系统中找到需要的binlog重新放回备库。

查看binlog有哪些文件用`show binary logs;`

加入临时实例需要的binlog从master.000005开始，备库上最早的binlog文件是master.000007

1. 从binlog备份下载缺失的两个文件，放到备库binlog目录
2. 打开binlog目录下的master.index(名字可能不一样，本机是binlog.00000x和binlog.index)，在文件开头加入`./master.000005` 和`./master.000006`
3. 重启备库让备库重新识别两个binlog文件
4. 临时实例和备库建立主备关系，正常同步

master-slave方案和mysqlbinlog方案都要求定期备份全量日志，且确保binlog在从本地删除前已经做了备份。

#### 延迟复制备库

假如一个库备份特别大，或者误操作的时间距离上一个全量备份的时间较长，那么恢复时间可能以天计。

如果有不允许太长恢复时间的业务，可以考虑搭建延迟复制的备库，mysql5.6引入这个功能。

延迟复制的备库是一种特殊的备库，通过`CHANGE MASTER TO MASTER_DELAY = N` 可以指定这个备库持续保持跟主库有N秒的延迟。

假如主库有数据被误删，在N秒之内发现，误删的命令在延迟复制的备库就还没执行。到备库执行`stop slave` 再通过前文方法，跳过误操作命令，就可以恢复出需要的数据。

#### 预防误删库/表的方法

账号分离。只给业务开发DML权限，不给truncate/drop权限，DDL需求通过管理系统支持。DBA日常也只使用只读账号，必要时才使用更新权限的账号。

制定操作规范。如删表前先对表做改名，观察一段时间，无影响再删除。改表名时加固定后缀，删表必须通过管理系统执行，只能删固定后缀的表。

### rm删除整个mysql

只是删除集群中某个节点，HA系统会选出新的主库。只需要在这个节点上恢复数据，再接入集群即可。

尽量把备份跨机房，或者跨城市。

## 32 kill不掉的语句

两种kill

- kill query + 线程id，终止这个线程正在执行的语句
- kill connection + 线程id，connection可省略，断开线程的连接

`kill connection` 只是把客户端的连接断开，后面的执行流程还是走`kill query` ，不同点在于`kill connection` 会把线程设置为“被`kill connection`” 的状态，`show processlist` 时检测到这个状态会显示`killed` ，而`kill query` 不会。

### 收到kill，线程做什么

用户执行`kill query thread_id_x` 时，处理kill命令的线程做了：

1. 把session x的运行状态改成`THD::KILL_QUERY` ，即变量`killed` 赋值为`THD::KILL_QUERY`
2. 给session x的执行线程发信号

假如session x处于锁等待，仅设置线程状态，线程x不知道状态变化，还是会继续等待。发信号是让session x退出等待来处理`THD::KILL_QUERY` 状态。

- 一条sql语句执行过程中有多处“埋点”判断线程状态，发现线程状态处于`THD::KILL_QUERY` 才进入语句终止逻辑。

- 线程如果处于等待状态，必须是可以被唤醒的等待，否则无法执行到“埋点”。

### kill不掉的例子

原文例子在mysql8.0.25已经不适用，作者是从源代码分析kill不掉的原因，可能源代码已经改写。

#### kill无效的情况

- 线程没有执行到判断线程状态的逻辑。例如线程无法进入InnoDB，进入sleep轮询出不来；IO压力过大，IO函数一直不返回，不能及时判断线程状态
- 终止逻辑耗时较长，从`show processlist ` 看`Command=Killed` 需要等终止逻辑完成语句才算真正完成。例如超大事务执行期间被kill，需要等回滚；大查询回滚删临时文件需要等IO；DDL被kill删临时文件需要等IO

### kill不掉时的办法

- 如果mysql版本对应的行为是并发线程达到`innodb_thread_concurrency` 的值以至于被kill的线程进不去InnoDB无法判断状态和执行终止逻辑，就临时调大并发数或者停掉别的线程。
- 如果回滚逻辑受IO资源限制执行慢，减少系统压力让它加速

### 客户端的ctrl+c，库名表名补全，-quick参数

mysql是停等协议，在客户端线程执行的语句没返回时，继续往连接发命令是没用的。客户端只能操作客户端的线程，执行ctrl+c，是客户端另启连接，发送kill query

客户端和服务端建立连接时，客户端会提供本地库名和表名补全的功能：

1. 执行`show databases;`
2. 切到连接时指定的库，执行`show tables;`
3. 用以上结果构建本地哈希表

当一个库中表的个数非常多，第三步会花比较长时间。

连接参数`-A` 可以关闭这个功能。

mysql客户端发送请求后，接收服务端返回的方式有两种：

1. 本地缓存，开内存把结果存起来。API对应`mysql_store_result` 或类似名字的
2. 不缓存，读一个处理一个，API对应`mysql_use_result`

客户端默认第一种，`-quick` 或`-q` 用第二种。这时如果本地处理慢，服务端发结果会被阻塞，因此会让服务端变慢。

`-quick` 有三个效果：

1. 跳过表名库名自动补全
2. 防止查询结果太大时耗费本地较多内存，影响机器性能
3. 不会把执行命令记录到本地的命令历史文件

所以是让客户端变得更快。

## 33 大查询（如全表扫描）对内存、server层、InnoDB层的影响

### 全表扫描对server层的影响

客户端：

`mysql -h$host -P$port -u$user -p$pwd -e "select * from db1.t" > $target_file`

所谓服务端的“结果集”，指一块限定大小的内存。

服务端取数据和发数据的流程：

1. 获取一行，写到`net_buffer` ，这块内存大小由参数`net_buffer_length` 定义，默认16k
2. 重复1直到`net_buffer` 写满，调用网络接口发送
3. 发送成功，清空`net_buffer` ，重复1、2
4. 发送函数返回`EAGAIN` 或`WSAEWOULDBLOCK` 表示本地网络栈（socket send buffer）已写满，进入等待。直到网络栈可写，再继续发送。

![查询结果发送流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/33/1.png)

一个查询在发送过程中，占用的mysql内部的内存最大就是`net_buffer_length` ，不会达到全表大小

`socket send buffer` 也不会达到全表大小，默认是`/proc/sys/net/core/wmem_default` ，如果写满，服务端会暂停读数据的流程

mysql是边读边发的，如果客户端接收慢，会导致服务端由于结果发不出去，事务的执行时间变长。

`show processlist ` 中`State` 一直显示`Sending to client` 表示服务器端的网络栈写满了。

如果要快速减少处于这个状态的线程，可以将`net_buffer_length` 设为更大的值，一般来说，`socket send buffer` 是几M，`net_buffer_length` 最大是1G，对执行器来说，只要能缓存在`net_buffer` 中，就已经算写出去了。虽然还是显示`Sending to client` ，但是语句已经执行完了，不会再占用资源比如锁。

`show processlist` 中显示`Sending data` 状态不一定指正在发送数据，可能处于执行器过程中的任意阶段。仅当一个线程处于"等待客户端接收结果"才会显示`Sending to client`

### 全表扫描对InnoDB的影响

内存的数据页由Buffer Pool（BP）管理，BP可以加速查询。一个查询的需要的数据页如果在内存里，就直接返回内存最新的结果，不需要读磁盘和应用redo log。

BP对查询的加速效果由内存命中率表示

`show engine innodb status` 中的`Buffer pool hit rate` 显示当前的内存命中率。一个稳定服务的线上系统，要保证响应时间符合要求，命中率要在99%以上。

InnoDB Buffer Pool的大小由参数`innodb_buffer_pool_size` 确定，一般设置成可用物理内存的60%～80%

InnoDB内存管理使用LRU（Least Recently Used）算法，核心是淘汰最久未使用的数据，使用链表实现。

基本的LRU把最近使用的数据页放链表头，最久未使用的放tail，空间不够时，新读入的数据页放链表头，淘汰tail的数据页。这样会产生读平时没有业务访问的历史数据表时，buffer pool里全是历史数据，正常的业务内存命令率急剧下降的问题。

![改进LRU算法](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/33/2.png)

InnoDB对LRU做了改进。

按5:3的比例把整个LRU链表分为young和old区域。LRU_old指向old区域第一个位置，整个链表的5/8处。

当访问不存在于链表的数据页时，淘汰tail，新插入的数据页放在LRU_old。

处于young区域的数据页，被访问时正常移动到链表头。

处于old区域的数据页，被访问时

- 这个数据页在LRU链表中存在超过1秒，就移动到链表头
- 短于1秒，位置保持不变。1秒是由参数`innodb_old_blocks_time` 指定，默认1000，单位毫秒。

此时如果还是全表扫描平时业务不会访问的历史数据表，新插入的数据页都放到old区域，且是顺序扫描，第一次和最后一次访问同一个数据页时间不超过1秒，不会移出old区，之后由于不再访问，被淘汰出链表。young区域可以响应正常业务，保证buffer pool的内存命中率。

## 34 可不可以使用join（join是怎么执行的、可能的问题、哪个表做驱动）

### join是怎么执行的

```mysql
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;
 
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();
 
create table t1 like t2;
insert into t1 (select * from t2 where id<=100)
```

#### Index Nested-Loop Join

`select * from t1 straight_join t2 on (t1.a=t2.a);`

防止mysql优化，这里用`straight_join` ，t1是驱动表，t2是被驱动表。

```mysql
mysql> explain select * from t1 straight_join t2 on (t1.a=t2.a);
+----+-------------+-------+------------+------+---------------+------+---------+------------------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref              | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------------------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | a             | NULL | NULL    | NULL             |  100 |   100.00 | Using where |
|  1 | SIMPLE      | t2    | NULL       | ref  | a             | a    | 5       | crashcourse.t1.a |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+------+---------+------------------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

```

join用上了被驱动表t2的索引a。

1. 遍历t1，从表t1读入一行R
2. 从R中取出a字段到t2去查找
3. 取出t2中满足条件的行，跟R组成一行，作为“结果集”的一部分
4. 重复1到3，直到遍历t1结束

这个过程形式上跟写程序时的嵌套查询类似，并且可以用上被驱动表的索引，称为Index Nested-Loop Join，简称NLJ

![NLJ执行流程图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/34/1.png)

1. 对t1全表扫描，要100行
2. 根据t1的a字段去t2查找走的是树搜索，每次扫描1行，总共100行
3. 一共扫描200行

##### 用join有更好吗？

如果不用join，使用单表查询

1. `select * from t1` ，100行
2. 遍历100行
   1. 每一行R取出a的值$R.a
   2. `select * from t2 where a=$R.a;`
   3. 返回结果和R构成结果集的一行

扫描200行，总共执行101条语句，比join多100次交互，还要自己拼接结果。join更好。

##### 选择驱动表

在这个join语句，驱动表走全表扫描，被驱动表走树搜索。

假设被驱动表行数M，每次树搜索的近似复杂度未$$\log_2{M}$$ ，索引a搜索一次，回表到主键索引搜索一次，因此被驱动表查找一行的时间复杂度为$$2*\log_2{M}$$

驱动表行数N，顺序扫描。每一行到被驱动表搜索一次。

整个执行过程，近似复杂度$$N+N*2*\log_2{M}$$

N对复杂度影响更大，因此可以使用被驱动表的索引时，应该小表做驱动表。

#### Simple Nested-Loop Join

`select * from t1 straight_join t2 on (t1.a=t2.b);`

按NLJ的流程，b上没有索引，每一次从t1取a字段匹配，都要全表扫描，因此一共扫描100*1000=100000行。

mysql没有用这个算法，用了“Block Nested-Loop Join”算法，简称BNL

#### Block Nested-Loop Join

被驱动表上没有可用的索引，流程如下：

1. 把t1的数据读入线程内存`joib_buffer` 中，由于是`select *` ，把整个t1表放入内存
2. 扫描t2，把每一行取出，跟`joib_buffer` 中的数据对比，满足join条件的，作为结果集的一部分返回

作者最新版本是8.0.13，而mysql8.0.18后增加了hash join，在8.0.25版本实验，explain结果是使用hash join

作者explain结果：

![不使用索引字段join的explain结果](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/34/2.png)

可以看到对t1和t2都做了全表扫描，总扫描行数1100。`join_buffer` 以无序数组方式组织，对t2每一行，都要做100次判断，总共在内存中做100*1000=100000次判断。

从时间复杂度来说，跟Simple Nested-Loop Join一样，但是BNL在内存操作，速度更快性能更好。

假设小表行数N，大表行数M，这个算法两个表都全表扫描，总扫描行数M+N，内存里判断次数是M*N，因此，**在`join_buffer` 可以把任意一个表都装入内存的情况下**，选择驱动表没有区别。

`join_buffer_size` 决定了`join_buffer` 的大小，默认256k。如果放不下t1的所有数据，就分段放。

> 把`join_buffer_size` 改成1200，执行过程变为：
>
> 1. 扫描t1，顺序读取数据放入`join_buffer` ，到第88行（作者说88是实际执行效果，也没说怎么看）buffer满了，到第2步
> 2. 扫描t2，逐行读取，跟`jpin_buffer` 中的数据对比，满足join条件的作为结果集的一部分返回
> 3. 清空`join_buffer` 
> 4. 继续扫t1，顺序读取最后12行数据放入buffer，继续第2步。

![BNL-两段](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/34/3.png)

4和5表示清空`join_buffer` 再复用。

由于t1分成两次放入join buffer，t2被扫描两次，在内存中判断等值条件的次数不变，88\*1000+12*1000=10万

假设驱动表行数N，分K段读入`join_buffer` ，被驱动表行数M。

N越大，K越大，因此把K表示为$$\lambda N,0<\lambda<1$$

因此扫描行数为$$N+\lambda NM$$ ，内存判断N*M次

考虑极限情况，M和N大小确定时，N小一些，结果更小。

因此，应该小表做驱动表。当N固定时，`join_buffer_size` 越大，K越小，全表扫描被驱动表的次数越少。所以join慢时，可以explain查看是否用BNL，可以尝试把`join_buffer_size` 改大。

##### 能不能用join

如果能用上被驱动表的索引，就是INLJ算法，可以。

如果explain里出现Block Nested-Loop，尽量不要用。大表join操作可能扫描被驱动表很多次，可能占用大量磁盘IO。

##### 用join的话谁做驱动表

总是应该小表做驱动表。在INLJ算法，选小表。在BNLJ算法，`join_buffer_size` 足够大时，都一样；不够大时，选小表。

where会决定谁是小表，两个表按照各自的条件过滤，过滤完成后计算参与join的各个字段的总数据量，数据量小的表是“小表”。

#### hash join

跟BNLJ不同的是，放入`join_buffer` 的是小表的hash表，key是on的条件，值是查询需要的列，小表的判断依据，跟上述相同。

如果`join_buffer` 能放入小表hash表的全部内容，称为CHJ（classic hash join）。如果放不下，参考https://www.cnblogs.com/cchust/p/11961851.html

> 在MySQL8.0中，如果join需要内存超过了join_buffer_size，build阶段会首先利用hash算将外表进行分区，并产生临时分片写到磁盘上；然后在probe阶段，对于内表使用同样的hash算法进行分区。由于使用分片hash函数相同，那么key相同(join条件相同)必然在同一个分片编号中。接下来，再对外表和内表中相同分片编号的数据进行CHJ的过程，所有分片的CHJ做完，整个join过程就结束了。

## 35 join优化

```mysql
create table t1(id int primary key, a int, b int, index(a));
create table t2 like t1;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t1 values(i, 1001-i, i);
    set i=i+1;
  end while;
   
  set i=1;
  while(i<=1000000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
 
end;;
delimiter ;
call idata();
```

### Multi-Range Read 优化（MRR）

目的是尽量使用顺序读盘。

`select * from t1 where a>=1 and a<=100;`

![基本回表流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/35/1.png)

如果随着a的值递增顺序回表查询，id值就变成随机的，出现随机访问，性能相对较差。

应用场景中大多数数据都是按照住建递增顺序插入得到的，所以可以认为，如果按照主键的递增顺序查询的话，对磁盘读比较接近顺序读，能够提升性能。这就是MRR的优化思路。

执行流程变为：

1. 根据索引a，定位到满足条件的记录(a,id)，将id值放入read_rnd_buffer
2. 将read_rnd_buffer中的id递增排序
3. 排序后的id数组依次到主键索引查找记录并作为结果返回

`read_rnd_buffer` 大小由参数`read_rnd_buffer_size` 控制。步骤1中buffer放满了就先执行2、3，然后清空buffer，执行1。

稳定使用MRR优化需要`set optimizer_switch="mrr_cost_based=off";`

![MRR执行流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/35/2.png)

```mysql
mysql> set optimizer_switch="mrr_cost_based=off";
Query OK, 0 rows affected (0.00 sec)

mysql> explain select * from t1 where a>=1 and a<=100;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                            |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------+
|  1 | SIMPLE      | t1    | NULL       | range | a             | a    | 5       | NULL |  100 |   100.00 | Using index condition; Using MRR |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+----------------------------------+
1 row in set, 1 warning (0.00 sec)

```

由于在`read_rnd_buffer` 中按id做了排序，最后得到的结果集也是按照主键id递增顺序的，与不用MRR时相反。

### Batched Key Access

mysql5.6后引入Batched Key Access（BKA）算法，是对NLJ的优化。

(I)NLJ的逻辑是从驱动表t1一行行取出字段a的值，再到被驱动表的索引查找主键id，再到主键id索引取出记录做join。对t2来说每次都匹配一个值，MRR的优势用不上。

可以一次把更多t1的行取出来，放到`join_buffer` 中，再传给t2，发挥顺序读的优势。

![BKA执行流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/35/3.png)

`join_buffer` 中放P1～P100表示不会把t1的整行放进去，只取查询需要的字段。如果`join_buffer` 放不下，会把数据分成多段执行上图流程。

要使用BKA，在执行sql前，先

`set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';`

BKA依赖于MRR，因此前两个参数先启动MRR。

### BNL的性能问题

使用BNL时可能对被驱动表做多次全表扫描，如果被驱动表是一个大的冷数据表：

- 冷表的数据量小于InnoDB的Buffer Pool的3/8，多次扫描冷表，且语句执行之间超过1秒，就会在再次扫描冷表的时候，把那个数据页移动到LRU的young区域头部
- 冷表很大，由于join语句在循环读冷表磁盘页和淘汰内存页，使得进入old区域的数据页很可能在1秒内被淘汰。此时一个正常业务要访问的数据页进入old区域，没有机会进入young，young区域的数据页也没有被合理地淘汰掉。

大的冷表join操作对IO有影响，但是在语句执行结束后，对IO的影响也结束；而对Buffer Pool的影响是持续性的，需要后续正常的业务查询请求慢慢恢复内存命中率。可以考虑增大`join_buffer_size` 的值减少被驱动表的扫描次数。

BNL对系统的影响主要是三方面：

1. 可能多次扫描被驱动表，占用磁盘IO
2. 判断join条件需要MN次对比（M、N是两张表行数），如果是大表会占用非常多cpu
3. 可能导致buffer pool的热数据被淘汰以及不需要的数据没有及时淘汰掉，影响内存命中率。

### BNL转BKA

一些情况下，直接在被驱动表上join条件的字段建索引，就可以把BNL转为BKA。

但如果被驱动表是大表，经过where过滤后，参与join的只有相对少量的数据，同时这条sql是低频语句，此时创建索引就浪费了，如

`select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;`

#### 以下内容可能对8.0.13前版本成立，对8.0.25不成立

如果用BNL算法执行上述语句，流程：

1. 把t1所有字段取出，存入`join_buffer` ，t1只有1000行，`join_buffer_size` 默认256k，可以存入。
2. 扫描t2，取出每一行跟`join_buffer` 中的数据对比
   - 不满足t1.b=t2.b，跳过
   - 满足，判断是否满足t2.b处于[1,2000]，是，作为结果集的一部分返回，否则跳过

此时判断次数是10亿次。explain可以看到使用了BNL，作者执行1分11秒。

可以考虑使用临时表优化。

1. 把t2满足条件的数据放在临时表tmp_t
2. 给tmp_t的字段b加索引
3. 让t1喝tmp_t做join操作，可以用BKA

对应的语句

```mysql
create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;
insert into temp_t select * from t2 where b>=1 and b<=2000;
select * from t1 join temp_t on (t1.b=temp_t.b);
```

我的理解，如果在t2上加索引再删索引，需要两次全表扫描，以及磁盘IO，比起加索引，建临时表的开销更小。

总体的思路都是让join能用上被驱动表的索引，触发BKA，提升查询性能。

这里用内存临时表效果更好(`engine=memory` )，原因有三个：

1. 比起InnoDB，Memory表不需要写磁盘，往`temp_t` 写数据速度更快
2. 临时表的索引b使用hash索引，查找速度比B+树更快
3. 临时表数据只有2000行，占用内存有限。

#### 8.0.25执行的效果

在8.0.25下执行相同语句，使用的是hash join

```mysql
mysql> explain select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra                                      |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   1000 |   100.00 | Using where                                |
|  1 | SIMPLE      | t2    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 998222 |     1.11 | Using where; Using join buffer (hash join) |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+--------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

mysql> select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;

.......
|  998 |    3 |  998 |  998 |  998 |  998 |
|  999 |    2 |  999 |  999 |  999 |  999 |
| 1000 |    1 | 1000 | 1000 | 1000 | 1000 |
+------+------+------+------+------+------+
1000 rows in set (0.19 sec)

```

### 扩展-hash join

`join_buffer` 里不是一个无序的数组，而是一个哈希表。那么就不需要10亿次判断，而是100万次hash查找。

原作者写时mysql不支持hash join，现在已经支持。

如果是不支持hash join的mysql版本，自己在业务端实现流程大致如下：

1. `select * from t1;` 取得t1的全部1000行，在业务端存入一个hash结构，如c++的set，php的dict
2. `select * from t2 where b>=1 and b<=2000;` 获得t2满足条件的2000行
3. 把这两千行取到业务端，逐行到hash结构的数据表匹配，满足条件的就join，输出到结果集。

## 36 为什么临时表可以重名？

- 内存表指memory引擎的表。数据都保存在内存，系统重启时清空，但表结构还在。(8.0以上版本临时表默认引擎为TempTable)
- 临时表可以用各种引擎，如果用InnoDB或者MyISAM，写数据时是写到磁盘上。

### 临时表的特性

1. 建表语法是`create temporary table ...`
2. 临时表只能被创建它的session访问，对其他线程不可见。
3. 可以与普通表同名。
4. session内有同名的普通表和临时表时，`show create table` 和增删改查访问的是临时表。
5. `show tables` 不显示临时表。
6. 创建临时表的session结束时，临时表会被自动删除。

由于特性6，临时表特别适合低频地从大表中取少部分数据来join，并且用不上索引的场景，比如第35章100万数据取2000行且join判断条件上被驱动表无索引。原因有2:

1. 不同session临时表可以同名，多个session同时执行join优化不担心建表失败
2. 临时表会自动回收，即使连接异常断开或数据库异常重启，也不需要额外删除查询中间过程需要的数据表。

### 临时表的应用

典型的场景如分库分表系统的跨库查询。

一般分库分表的场景就是把一个逻辑上的大表分散到不同的数据库实例上的不同表上。

比如将大表ht按照字段f拆分成1024个分表，分布到32个数据库实例上。

![分库分表简图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/36/1.png)

一般都有proxy层。

这个架构中，分区key选择的依据是“减少跨库和跨表查询”。如果大部分语句都包含f的等值条件，就用f做分区键，这样proxy解析完sql后就能确定将语句路由到哪个分表。

比如`select v from ht where f=N;`

上述语句可以用分表规则（如N%1024）确认需要的数据在哪个分表上。

如果表上还有另一个索引k，查询语句

`select v from ht where k>=M order by t_modified desc limit 100;`

 由于查询条件里没有分区字段f，只能到所有分区中查找满足条件的行然后统一order by。这种情况有两种常用思路：

一是在proxy层的进程代码中排序。优势是处理速度快，proxy层拿到分库的数据后直接在内存中计算。缺点是中间层的开发工作量大，以及对proxy端的性能压力笔记大，很容易出现内存和cpu不够用的问题。

二是把各个分库拿到的数据汇总到一个mysql实例的一个表，在这个实例上做逻辑操作。

比如上述语句可以：

- 汇总库上创建临时表`temp_ht` ，表里包含`v,k,t_modified`
- 在各个分库上执行`select v,k,t_modified from ht_x where k>=M order by t_modified desc limit 100;`
- 分库执行结果插入`temp_ht`
- 执行`select v from temp_ht order by t_modified desc limit 100;`

![跨库查询流程示意图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/36/2.png)

实践中往往发现每个分库计算量都不饱和，所以会直接把`temp_ht` 放在32个分库中的某一个上。

### 为什么临时表可以重名

#### 以下在8.0.25已经不成立

执行`create temporary table temp_t(id int primary key)engine=innodb;` 时mysql要给这个表创建一个frm文件保存表结构定义，还要有地方存数据。

frm放在临时文件目录下，路径用`select @@tmpdir` 查看，文件后缀是frm，前缀是`#sql{进程id}_{线程id}_序列号` 。

表数据的存放方式：

- 5.6以及之前，mysql在临时文件目录下创建一个相同前缀，ibd为后缀的文件存放数据
- 从5.7开始，mysql引入临时文件表空间，专门存临时文件的数据，不需要创建ibd文件。

从文件名可以看到，mysql在存储上认为临时表的表名和普通表的表名是不同的，因此两者可以在同一个库下重名。

mysql维护数据表除了物理上要有文件外，内存里也有一套机制区别不同的表，每个表对应一个`table_def_key`

- 普通表的`table_def_key` 又库名+表名得到
- 临时表由库名+表名+`server_id` + `thread_id`

因此两个会话创建的同名临时表的`table_def_key` 不同，磁盘文件名也不同，因此可以并存。

每个线程都维护自己的临时表链表。session操作表时先遍历链表看是否有临时表，有则优先操作，否则再操作普通表。session结束时对链表里的每个临时表执行`drop temporary table tbl_name` 并且把这条命令记录到binlog(前提是statement/mixed格式)。

#### 8.0.25临时表

表的结构和数据存储在mysql的data目录下的`#innodb_temp` 目录中，以ibt为后缀，会话终止空间就被回收。mysql的data目录下的`ibtmp1` 文件存储临时表的回滚日志，在mysql服务器重启时空间才回收。

### 临时表和主备复制

binlog如果为row格式，记录日志时会记录数据，此时临时表相关的语句就不会记录到binlog里。

binlog为statement/mixed格式时，binlog才会记录临时表的操作。这时创建临时表的操作会传到备库执行，主库的线程退出时，自动删除临时表，但备库的同步线程是持续运行的，所以需要在binlog里记录删除的语句传给备库执行。

mysql记录binlog时还会记录执行这个语句的线程id。备库的同步线程就可以知道执行每个语句的主库线程id，并利用这个线程id构造临时表的`table_def_key` ：

1. 主库上session A构建的临时表t1，在备库的`table_def_key` 是库名+t1+备库的`server_id` +session A的`thread_id`
2. 主库上session B构建的临时表t2，在备库的`table_def_key` 同理

由于`table_def_key` 不同，这两个表在备库的应用线程里不会冲突。

## 37 mysql什么时候会使用内部临时表

### union执行流程

```mysql
create table t1(id int primary key, a int, b int, index(a));
delimiter ;;
create procedure idata()
begin
  declare i int;
 
  set i=1;
  while(i<=1000)do
    insert into t1 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

explain (select 1000 as f) union (select id from t1 order by id desc limit 2);
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------+
| id | select_type  | table      | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                            |
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------+
|  1 | PRIMARY      | NULL       | NULL       | NULL  | NULL          | NULL    | NULL    | NULL | NULL |     NULL | No tables used                   |
|  2 | UNION        | t1         | NULL       | index | NULL          | PRIMARY | 4       | NULL |    2 |   100.00 | Backward index scan; Using index |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL   | NULL          | NULL    | NULL    | NULL | NULL |     NULL | Using temporary                  |
+----+--------------+------------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------+
3 rows in set, 1 warning (0.01 sec)

```

这条语句的执行流程是：

1. 创建一个内存临时表，只有一个整型字段f，且f是主键
2. 执行第一个子查询，得到1000，存入临时表中
3. 执行第二个子查询：
   - 拿到第一行id=1000，试图插入临时表中，由于1000已经存在，违反唯一性约束，插入失败，继续执行
   - 拿到第二行id=999，插入临时表成功

4. 从临时表中按行取数据，返回结果，并删除临时表。

临时表起暂存数据的作用，计算过程还用上了临时表主键的唯一性约束，实现union的语义。

如果`union` 改成`union all` ，就没有去重语义。执行的时候依次执行子查询，得到的结果直接作为结果集的一部分发给客户端，不需要临时表。

```mysql
explain (select 1000 as f) union all (select id from t1 order by id desc
limit
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                            |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------+
|  1 | PRIMARY     | NULL  | NULL       | NULL  | NULL          | NULL    | NULL    | NULL | NULL |     NULL | No tables used                   |
|  2 | UNION       | t1    | NULL       | index | NULL          | PRIMARY | 4       | NULL |    2 |   100.00 | Backward index scan; Using index |
+----+-------------+-------+------------+-------+---------------+---------+---------+------+------+----------+----------------------------------+
2 rows in set, 1 warning (0.00 sec)

```

### group by执行流程

```mysql
select id%10 as m, count(*) as c from t1 group by m;
+------+-----+
| m    | c   |
+------+-----+
|    1 | 100 |
|    2 | 100 |
|    3 | 100 |
|    4 | 100 |
|    5 | 100 |
|    6 | 100 |
|    7 | 100 |
|    8 | 100 |
|    9 | 100 |
|    0 | 100 |
+------+-----+
10 rows in set (0.01 sec)

explain select id%10 as m, count(*) as c from t1 group by m;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+------------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                        |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+------------------------------+
|  1 | SIMPLE      | t1    | NULL       | index | PRIMARY,a     | a    | 5       | NULL | 1000 |   100.00 | Using index; Using temporary |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+------------------------------+
1 row in set, 1 warning (0.00 sec)
```

- Using index表示使用覆盖索引，选择索引a，不需要回表。
- Using temporary表示使用临时表

原作者的结果还有Using filesort，需要排序。实测8.0.25版本不需要排序，即默认`order by null`

执行流程：

1. 创建内存临时表，有两个字段m和c，主键m
2. 扫描t1的索引a，依次取出叶节点上的id值，计算id%10的结果，这里记做x
   - 临时表中没有主键为x的行，插入记录(x,1)
   - 否则将x这一行的c值加1

3. 遍历完成后，根据字段m做排序，得到结果集返回客户端（8.0.25版本不排序）

内存临时表的大小由`tmp_table_size` 控制，默认16M。

如果临时表的数据量大，内存放不下，会把内存临时表转成磁盘临时表，默认用InnoDB引擎。

#### group by优化方法--索引

group by的语义这里是统计不同的值出现的个数，由于每一行的id%10的结果无序，所以需要临时表来记录并统计结果。

如果扫描过程中保证出现的数据有序：

$$\overbrace{0,0,\cdots,0}^X\overbrace{1,1,\cdots,1}^Y\overbrace{2,2,\cdots,2}^Z$$

从左到右顺序扫描，依次累加，碰到第一个1，已经知道累积X个0，结果集第一行就是(0,X)。

按照这个逻辑，扫描到输入数据结束，就可以拿到group by结果，不需要临时表和排序。

InnoDB的索引就可以满足输入有序的条件。

```mysql
mysql> alter table t1 add column z int generated always as(id % 100), add index(z);
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> explain select z, count(*) as c from t1 group by z;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | index | z             | z    | 5       | NULL | 1000 |   100.00 | Using index |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

```

此时索引z上的数据就是类似X个0Y个1那样有序的，可以看到不需要临时表了。

#### group by优化方法--直接排序

如果一开始就知道一个group by需要放到临时表的数据量特别大，可以直接用磁盘临时表。

在`group by` 中加入`SQL_BIG_RESULT` 这个提示可以告诉优化器语句涉及的数据量很大，直接用磁盘临时表。

这时磁盘临时表从磁盘空间（存储效率）考虑，不用B+树，而用数组。

`select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;`

执行流程：

1. 初始化`sort_buffer` 确定放入一个整型字段，计为m
2. 扫描t1的索引a，依次取出id值，将id%100存入`sort_buffer`
3. 扫描完成，对`sort_buffer` 的字段m排序；如果`sort_buffer` 内存不够用，就会用磁盘临时表辅助排序。
4. 排序完成，得到有序数组。
5. 如同上一节扫描索引z，顺序扫描有序数组，直接得到结果。

![使用SQL_BIG_RESULT的执行流程图](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/37/1.png)

执行下列语句前需要先把z列删掉。

```mysql
explain select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by
 m;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                       |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------------+
|  1 | SIMPLE      | t1    | NULL       | index | PRIMARY,a     | a    | 5       | NULL | 1000 |   100.00 | Using index; Using filesort |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)

```

可以看到没有再使用临时表存数据而是直接使用排序算法。

### mysql什么时候用内部（内存）临时表

1. 如果语句执行过程可以一边读数据一边直接得到结果，就不需要额外内存，否则需要额外内存来保存中间结果
2. `join_buffer` 是无序数组，`sort_buffer` 是有序数组，临时表是二维表结构。
3. 执行逻辑需要用到二维表特性就会优先考虑用临时表。比如union需要用到唯一性约束，group by需要用到另一个字段存聚合函数的值。

### group by的使用指导原则

1. 如果对group by的结果没有排序要求，在语句后加`order by null` （8.0.25已经默认不排序）
2. 尽量让group by用上表的索引，确认方法是explain结果里**没有** Using temporary和Using filesort
3. group by需要统计的数据量不大，尽量只用内存临时表；可以适当调大`tmp_table_size` 避免用磁盘临时表
4. 如果预见到数据量内存临时表放不下，使用`SQL_BIG_RESULT` 提示，告诉优化器不通过临时表记录group by的中间数据，而是直接把聚合函数计算值排序成有序数组，通过顺序读取来得到结果。(如果没有使用这个提示，如果group by要用到磁盘临时表，默认引擎是InnoDB，而InnoDB的索引是有序的，主键索引一般就是group条件的计算值，所以读到的group by结果也是有序的，逻辑跟顺序读有序数组类似。在8.0.25不成立，即便用到磁盘临时表，group by也没有排序，原因待探究。默认引擎仍然是InnoDB，但是没有按group条件排序，所以可能是主键发生了变化。)

## 38 InnoDB跟Memory引擎对比

8.0.25中，内存临时表已经默认使用TempTable引擎。

### 内存表的数据组织结构

```mysql
create table t1(id int primary key, c int) engine=Memory;
create table t2(id int primary key, c int) engine=innodb;
insert into t1 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
insert into t2 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
```

![t1的数据组织结构](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/38/1.png)

内存表的数据部分以数组的形式单独存放，主键id索引是hash索引，实际上应该是hash(1),hash(0)...，索引上的key并不是有序的，值是每个数据的位置（指针？）。

在t1中执行`select *` ，是顺序扫描整个数组，全表扫描，并不按主键索引的顺序。

InnoDB和Memory的数据组织方式不同：

1. InnoDB把数据放在主键索引上，其他索引上保存的值是主键id。这种方式称为索引组织表。
2. Memory把数据单独存放，索引上保存数据的位置。这种方式称为堆组织表。

从中可以看出两个引擎的一些典型不同：

1. InnoDB表的数据总是有序存放，Memory的数据按照写入顺序存放。
2. 数据文件有空洞时，InnoDB为了保证新插入数据的有序性，只能在固定位置写入新值，内存表有空位就可以插入新值。
3. 数据存放的位置发生变化时，InnoDB只需要改主键索引，Memory需要改所有索引
4. InnoDB用主键索引查询时需要走一次索引查找，用普通索引查询时要走两次索引查找（这里不考虑覆盖索引）。内存表都只走一次索引查找，所有索引的地位相同。
5. InnoDB支持变长数据类型，不同记录的长度可能不同。Memory不支持Blob和Text字段，并且即使定义`varchar(N)` 实际也当作`char(N)` 也就是固定长度字符串来存储，因此Memory表的每行数据长度相同。（不支持变长类型，在TempTable引擎中已经支持。）

Memory表的主键索引是哈希索引，所以范围查询用不上索引，需要全表扫描。

### hash索引和B-Tree索引

内存表可以通过B-Tree索引来支持范围查询。

`alter table t1 add index a_btree_index using btree (id);`

此时t1的数据组织结构如下：

![增加BTree索引的t1的数据组织结构](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/38/2.png)

这个B-Tree索引跟InnoDB的B+树索引组织形式类似。

范围查询时，优化器会选择btree索引，避免全表扫描。

```mysql
mysql> select * from t1 where id < 5;
+----+------+
| id | c    |
+----+------+
|  0 |    0 |
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
5 rows in set (0.00 sec)

mysql> select * from t1 force index(primary) where id < 5;
+----+------+
| id | c    |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
|  0 |    0 |
+----+------+
5 rows in set (0.00 sec)
```

不建议生产环境上用Memory表，原因主要是锁粒度问题和数据持久化问题。

### Memory表的锁

原作写的是内存表的锁，但是在8.0.25，内存表的引擎多了一个TempTable，为了严谨这里写Memory表。

不支持行锁，只支持表锁。一张表只要有更新，就会堵住其他所有这个表上的读写操作。

因此内存表的锁粒度决定了处理并发事务时性能不好。

### Memory表数据持久性问题

数据库重启时所有的内存表都清空。

#### M-S架构下内存表的问题

![M-S基本架构](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/38/3.png)

假如有以下时序：

1. 业务正常访问主库
2. 备库硬件升级重启，内存表t1内容被清空
3. 备库重启后客户端发送update修改t1的行，备库应用线程报错“找不到要更新的行”，导致主备同步停止。

如果这时主备切换，客户端会看到t1数据丢失。

在有proxy的架构里大家默认主备切换的逻辑由数据库系统自己维护，对客户端现象就是连接断开重连后数据丢失。

mysql为了防止主库重启后，内存表数据丢失，造成主备不一致，在数据库重启后，往binlog写入一行`DELETE FROM t1`

如果使用双M架构，备库重启后上述delete会传到主库，把主库内存表的内容删除。

### 使用Memory表的场景

普通的内存表建议都用InnoDB表来代替。

1. 如果表更新量大，并发度是很重要的参考指标，innoDB支持行锁，并发度更好
2. 内存表的数据量不大，如果考虑读性能，读QPS很高且数据量不大的用，InnoDB会缓存在InnoDB Buffer pool的LRU链表中，内存命中率高，性能也不会差。

有一个场景例外，如35、36章用到的用户临时表（join语句优化，`create temporary table...`），在数据量可控时可以考虑内存表。

在join语句优化时，创建的临时表不会被其他线程访问，没有并发度问题；重启后本身就要删除，持久性问题不存在；备库的临时表被delete也不会影响主库的用户线程（`table_def_key` 做区分）

## 39 自增主键为什么不连续

自增主键不能保证连续递增。

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;
```

### 自增值保存在哪

表的结构定义存放在后缀名为frm的文件中，不会保存自增值。（mysql8以后，取消frm文件，表数据存在ibd后缀文件里，表的元数据如结构等保存在系统表空间里，系统表空间是ibdata以及mysql.ibd）

不同引擎对自增值的保存策略不同

- MyISAM引擎的自增值保存在数据文件里
- InnoDB的自增值（AUTO_INCREMENT）保存在内存里，在8.0版本以后，才有自增值持久化的能力。
  - 在5.7及之前，自增值保存在内存里，每次mysql重启后，第一次打开表时会去找自增值的最大值`max(id)` ，将`max(id)+1` 作为这个表当前的自增值，即mysql重启可能会修改一个表的自增值。（重启前`AUTO_INCREMENT=11`  ，删除id=10的行，重启后`AUTO_INCREMENT=10` ）。
  - Mysql8.0将自增值的变更记录在redo log中，重启的时候依靠redo log恢复重启前的值。

### 自增值修改机制

如果字段id定义为AUTO_INCREMENT，插入数据时：

1. 插入时id指定为0，null或未指定，就把这个表当前的AUTO_INCREMENT值（假设为Y）填到自增字段
2. 插入数据时id指定了具体值，就使用指定值X。如果X<Y，表的自增值不变。X≥Y，从`auto_increment_offset` 开始以`auto_increment_increment` 为步长，找到第一个大于X的值作为新的自增值。两个系统参数默认值都是1，表示自增的初始值和步长。

### 自增主键不连续的原因

1. 唯一键冲突
2. 事务回滚
3. 批量申请自增id但是没用上

### 自增值的修改时机

#### 唯一键冲突

```mysql
mysql> show create table t;
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                            |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| t     | CREATE TABLE `t` (
  `id` int NOT NULL AUTO_INCREMENT,
  `c` int DEFAULT NULL,
  `d` int DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
+-------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select * from t;
+----+------+------+
| id | c    | d    |
+----+------+------+
|  1 |    1 |    1 |
+----+------+------+
1 row in set (0.01 sec)

mysql> insert into t values(null, 1, 1);
ERROR 1062 (23000): Duplicate entry '1' for key 't.c'
mysql> insert into t values(null, 2, 2);
Query OK, 1 row affected (0.01 sec)

mysql> select * from t;
+----+------+------+
| id | c    | d    |
+----+------+------+
|  1 |    1 |    1 |
|  3 |    2 |    2 |
+----+------+------+
2 rows in set (0.00 sec)
```

`insert into t values(null,1,1);` 执行流程：

1. 执行器调用InnoDB接口写入这一行，传入的值是(0,1,1);
2. InnoDB发现用户没有指定自增id的值，获取表t当前的自增值2
3. 将传入的行改成(2,1,1)
4. 将表的自增值改成3
5. 真正执行插入操作，不满足唯一性约束，返回。

可以看到自增值改为3之后没有再改回2，因此再插入新行拿到的自增值就是3，出现了自增主键不连续。

#### 事务回滚

```mysql
mysql> select * from t;
Empty set (0.00 sec)

mysql> insert into t values(null, 1, 1);
Query OK, 1 row affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t values(null, 2 ,2);
Query OK, 1 row affected (0.00 sec)

mysql> rollback;
Query OK, 0 rows affected (0.01 sec)

mysql> insert into t values(null, 2 ,2);
Query OK, 1 row affected (0.00 sec)

mysql> select * from t;
+----+------+------+
| id | c    | d    |
+----+------+------+
|  4 |    1 |    1 |
|  6 |    2 |    2 |
+----+------+------+
2 rows in set (0.00 sec)
```

#### 自增值为什么不能回退

为了性能考虑。

假设可以回退：

假设有两个并行的事务，在申请自增值时，申请自增id要加锁顺序申请。假设：

1. 事务A申请到id=2，B申请到id=3，此时t的自增值为4
2. B正确提交，A唯一键冲突
3. A把自增id回退，改回2
4. 其他事务会申请到id=2，然后id=3，这时插入报错主键冲突

为了解决主键冲突，有两种方法：

1. 每次申请id前，先判断表里是否存在这个id，存在则跳过。本来申请id很快，现在还要去主键索引树上判断
2. 把自增id的锁范围扩大，必须等上一个事务执行完并提交，下一个事务才能再申请自增id，这样B申请到的id就是2。这个方法锁的粒度太大，系统并发能力下降。

因此为了性能，自增id不能回退。

### 自增锁的优化

mysql5.0，自增锁在一条语句执行结束才释放。

Mysql5.1.22，新增参数`innodb_autoinc_lock_mode` ，默认值1

1. 设为0，自增锁和5.0一样
2. 设为1
   - 普通insert语句，自增锁在申请自增值之后马上释放
   - 类似`insert...select...` 这样的批量插入语句，等语句执行结束才释放。类似的有`replace...select...` 和`load data` 
3. 设为2，所有的申请自增主键的动作都是申请后就释放。

生产上，尤其是有`insert...select...` 类似批量插入数据的场景，建议设置`innodb_autoinc_lock_mode=2` 和`binlog_format=row` ，既提升并发性，又不会出现数据一致性问题。

#### 批量申请自增值但是用不上

对于批量插入数据的语句，有对应的批量申请自增id的策略：

1. 语句执行过程中，第一次申请自增id，分配1个
2. 1个用完以后，第二次申请，分配2个
3. 2个用完以后，第三次申请，分配4个
4. 同一个语句申请自增id，每次申请到的个数是上一次的两倍

比如

```mysql
insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);
create table t2 like t;
insert into t2(c,d) select c,d from t;
insert into t2 values(null, 5,5);
```

`insert...select` 分3次申请自增id，第一次申请到id=1，第二次申请到id=2，3，第三次申请到id=4，5，6，7。实际上只用到4，5、6、7被浪费掉。最后一句实际插入的是(8,5,5)。

## 40 insert语句的锁

```mysql
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;
 
insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);
 
create table t2 like t;
```

### insert...select 语句

在可重复读下，`binlog_format=statement` 时执行`insert into t2(c,d) select c,d from t;` 会对表t的所有行和间隙加锁。（并不是一直都锁全表的意思，也是只锁住需要访问的资源，只是在这里刚好锁全表）

|                 A                 |                    B                     |
| :-------------------------------: | :--------------------------------------: |
| `insert into t values(-1,-1,-1);` | `insert into t2(c,d) select c,d from t;` |

如果B先执行，由于对t的主键索引加了$$(-\infin,1]$$ 的next-key lock，会在B执行完成后A才能insert

如果没有锁，可能B先执行，但是后写入binlog，在`binlog_format=statement` 时，主库的t2没有id=-1的行，而备库应用binlog同步出来的t2会有id=-1的行。

`insert into t2(c,d) (select c+1, d from t force index(c) order by c desc limit 1);`

这条语句的加锁范围是表t索引c上的(3,4]和(4,supremum]以及主键索引上id=4这一行。

执行流程是从表t中按照索引c倒序扫描第一行，拿到结果写到t2中，整条语句的扫描行数是1。

### insert循环写入

**以下内容8.0.25不成立**

`insert into t(c,d) (select c+1,d from t force index(c) order by c desc limit 1);`

原作者看慢日志显示`Rows_examined:5` ，explain结果select语句Using temporary，且explain里rows=1是受limit影响。

用`show status like '%innodb_rows_read%';` 查看，执行语句前后相差4行，因为临时表默认用Memory引擎，所以这4行是查t，即对t做了全表扫描。

执行过程：

1. 创建临时表，有字段c和d
2. 按照索引c扫描表t，依次取c=4、3、2、1并回表，读到整行的值写入临时表。这时`Rows_examined=4` 
3. 语句有limit 1，所以只取临时表第一行再插入表t中，这时`Rows_examined=5` 

这条插入语句导致表t做全表扫描，会给索引c上所有间隙都加上共享的next-key lock（读锁），所以这条语句执行期间其他事务不能在这个表上插入数据。

需要临时表的原因是这类一边遍历数据一边更新数据的情况，如果读出来的数据直接写回原表，就可能在遍历过程中，读到刚刚插入的记录，新插入的记录如果参与计算逻辑，跟语义不符。

由于实现上这个语句没有在子查询中直接使用limit 1，导致需要遍历整个表t。优化考虑使用内存临时表`temp_t` ，先`insert into` 到`temp_t` ，只需扫描一行，再从`temp_t` 取出这行插入表t。

```mysql
create temporary table temp_t(c int,d int) engine=memory;
insert into temp_t  (select c+1, d from t force index(c) order by c desc limit 1);
insert into t select * from temp_t;
drop table temp_t;
```

#### 8.0.25版本纠错

同样的语句，慢日志`Rows_examined:1` ，explain结果里select语句Extra段显示Backward index scan; Using temporary，rows=1

用`show status like '%innodb_rows_read%';` 查看，执行语句前后相差1行

加锁范围是表t索引c上(5,supremum]（写锁），(4,5]（写锁），(3,4]（读锁）以及表t主键id索引上c=4（读锁）、5（写锁）对应的行（正常的话id=4、5）。

需要临时表的原因，应该跟原作者说的一样。

扫描行数只有1行，是desc和limit起了效果，倒序扫描索引c，找到c最大的一行，之后我认为是插入了内存临时表（默认引擎Temptable），所以`show status like '%innodb_rows_read%';` 读出来行数是1，最后从内存临时表读出再插入表t。慢日志也显示`Rows_examined:1` 的原因可能跟内存临时表的处理有关，需要看源代码。

### insert唯一键冲突

在可重复读下

|                              A                               |                     B                     |
| :----------------------------------------------------------: | :---------------------------------------: |
|              `insert into t values(10,10,10);`               |                                           |
| `begin;`<br /> `insert into t values(11,10,10);` (Duplicate entry '10' for key 'c') |                                           |
|                                                              | `insert into t values(12,9,9);` (blocked) |

A执行insert时发生唯一键冲突，还是在冲突的索引上加了锁。原作者说的是加了(5,10]的读锁（5怎么来的？作者如果不在执行语句前展示表内容很容易疑惑），实测上述语句是被主键索引上的(10,supremum]锁住，如果B插入的是(6,9,9)，则会申请间隙锁的时候被阻塞，因此加锁范围是主键索引上(10,supremum]以及索引 c上(4,10]（读锁）（假设文章开头初始化表之后就没有操作）。

这里的加锁似乎没有合理的解释。

这里如果执行的语句改成`insert into t values(11,10,10) on duplicate key update d=100;` ，就会在索引c上加(4,10]的写锁。



|      |                          A                          |                  B                  |                  C                  |
| :--: | :-------------------------------------------------: | :---------------------------------: | :---------------------------------: |
|  T1  | `begin;` <br /> `insert into t values(null, 5, 5);` |                                     |                                     |
|  T2  |                                                     | `insert into t values(null, 5, 5);` | `insert into t values(null, 5, 5);` |
|  T3  |                     `rollback;`                     |                                     |           Deadlock found            |

1. T1时刻启动A并执行insert，此时在索引c的c=5上加记录锁。这个索引是唯一索引，因此退化为记录锁。
2. T2时刻B要执行相同的insert，发现唯一键冲突，加上next-key lock读锁（`show engine innodb status\G` 显示`lock mode S` ，没有其他信息，这里认为是next-key lock），(4,5]，同样C也在索引c上加读锁。（实际上加间隙锁不冲突，这里等待就是在加记录锁的时候被阻塞所以要等待，没有显示只需要加记录锁可能是因为还没到退化的时机？）
3. T3时刻，A回滚，这时候B和C都试图继续执行插入，都要加写锁。原作者说两个session都等待对方的行锁，所以出现死锁。我认为有误。8.0.25版本实测，`show engine innodb status` 查看死锁，两个session都持有(4,supremum]的读锁，并且都在申请(4,supremum]的写锁这一步被对方阻塞，因此出现死锁。

### insert into ... on duplicate key update

语义是插入一行数据，如果碰到唯一键约束，就执行后面的更新语句。

如果有多个列违反唯一性约束，就按照索引的顺序，修改跟第一个索引冲突的行。

```mysql
mysql> select * from t;
+----+------+------+
| id | c    | d    |
+----+------+------+
|  1 |    1 |    1 |
|  2 |    2 |    2 |
+----+------+------+
2 rows in set (0.00 sec)

mysql> insert into t values(2,1,100) on duplicate key update d=100;
Query OK, 2 rows affected (0.00 sec)

mysql> select * from t;
+----+------+------+
| id | c    | d    |
+----+------+------+
|  1 |    1 |    1 |
|  2 |    2 |  100 |
+----+------+------+
2 rows in set (0.00 sec)

```

主键id先判断，mysql认为这个语句跟id=2这一行冲突，所以修改id=2这一行。

真正更新的只有一行。代码实现上，insert和update都认为自己成功了，update计数加1，insert计数也加1。

## 41 怎么最快地复制一张表

```mysql
create database db1;
use db1;
 
create table t(id int primary key, a int, b int, index(a))engine=innodb;
delimiter ;;
  create procedure idata()
  begin
    declare i int;
    set i=1;
    while(i<=1000)do
      insert into t values(i,i,i);
      set i=i+1;
    end while;
  end;;
delimiter ;
call idata();
 
create database db2;
create table db2.t like db1.t;
```

假设现在要把`db1.t` 里a>900的数据导出插入到`db2.t` 

### mysqldump方法

使用mysqldumo命令将数据导出成一组insert语句

`mysqldump -h$host -P$port -u$user --add-locks=0 --no-create-info --single-transaction --set-gtid-purged=OFF db1 t --where="a>900" --result-file=/client_tmp/t.sql`

- `--single-transaction` 作用是导出数据时不需要对`db1.t` 加表锁，而是使用`start transaction with consistent snapshot` 的方法
- `--add-locks=0` 表示在输出的文件结果里，不增加"LOCK TABLES t WRITE"
- `--no-create-info` 表示不需要导出表结构
- `--set-gtid-purged=off` 表示不输出跟GTID相关的信息

结果的一条insert语句里会包含多个value对，为了后续用这个文件来插入数据时执行速度更快。单行的数据量不会超过`net_buffer_length` ，可以通过执行mysqldump时增加`--net_buffer_length` 控制。

在执行命令时加入`--skip-extended-insert` 可以让生成文件中一条insert只插入一行数据。

通过以下命令将生成的语句放到db2库执行

`mysql -h$host -P$port -u$user db2 -e "source /client_tmp/t.sql"`

source并不是sql语句而是客户端命令。

客户端执行source流程：

1. 打开文件，默认以分号为结尾读取一条条的sql语句
2. 将sql语句发到服务端执行

### 导出CSV文件

`select * from di1.t where a>900 into outfile '/server_tmp/t.csv';`

1. 语句的结果保存在服务端
2. 文件的生成位置受参数`secure_file_priv` 限制
   - 如果设置`empty` ，表示不限制文件生成的位置，不安全。
   - 设置为表示路径的字符串，表示生成的文件只能放在这个指定的目录和子目录下
   - 设为NULL，表示禁止在这个mysql实例上执行这个操作
3. 命令不会覆盖文件，如果已存在就报错

可以用以下命令导入`db2.t`

`load data infile '/server_tmp/t.csv' into table db2.t;`

1. 打开文件csv，以制表符作为字段分隔符，以换行符作为记录分隔符，读取数据
2. 启动事务
3. 判断每一行的字段数与`db2.t` 是否相同，不同则报错，事务回滚；相同则调用InnoDB引擎接口，写入表中。
4. 重复3直到csv文件读入完成，提交事务。

以上是主库上的执行流程。考虑主备同步，在`binlog_format=statement` 时，仅仅在binlog里写入load会由于备库没有csv文件导致主备同步停止。所以load的完整流程是：

1. 主库执行完成后将csv文件的内容直接写到binlog
2. 往binlog中写入`load data local infile '/tmp/SQL_LOAD_MB-1-0' INTO TABLE db2.t`
3. binlog传到备库
4. 备库的应用线程执行这个事务的日志时先将binlog中csv文件的内容读出，写到本地临时目录`/tmp/SQL_LOAD_MB-1-0` 中，再执行`load data` 

![loaddata同步流程](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/41/1.png)

`load data` 在不加local时，读服务端文件，这个文件必须在`secure_file_priv` 指定的目录或子目录下；加local时读客户端文件，mysql客户端先把本地文件传给服务端然后执行上述load data流程。

`select ... into outfile` 不会生成表结构文件。mysqldump提供`--tab` 参数可以同时导出表结构定义文件和csv数据文件。

`mysqldump -h$host -P$port -u$user --single-transaction --set-gtid-purged=OFF db1 t --where="a>900" --tab=$secure_file_priv`

这条命令会在`$secure_file_priv` 定义的目录下创建t.sql保存建表语句，一个t.txt保存数据

### 物理拷贝方法

**mysql8保存表结构和数据的方式跟5.x版本不同，以下内容可能在8不适用**

一个InnoDB表除了包含存储表结构的frm文件和存储数据的ibd文件外，还需要在数据字典中注册。直接拷贝`db1.t` 的物理文件到`db2` 目录下，因为数据字典中没有`db2.t` 表，系统不会识别和接受这两个物理文件。

mysql5.6引入可传输表空间（transportable tablesapce）的方法，通过导出+导入表空间的方式实现物理拷贝表的功能。

假设目标是在`db1` 库下复制一个跟表t相同的表r。

1. `create table r like t;` 创建相同表结构的空表
2. `alter table r discard tablespace` 删除`r.ibd` 
3. `flush table t for export` 锁表，在`db1` 目录下生成一个`t.cfg` 文件
4. 在`db1` 目录下执行`cp t.cfg r.cfg;cp t.ibd r.ibd` 
5. `unlock tables` ，`t.cfg` 会被删除
6. `alter table r import tablespace` ，将`r.ibd` 作为表r的新的表空间，由于`r.ibd` 和`t.ibd` 内容相同，表r中就有了跟表t相同的数据。注意mysql对物理文件要有读写权限。

第3步flush之后表t处于只读状态，执行unlock才释放读锁。

执行`import tablespace` 时为了让文件里的表空间id跟数据字典中的一致，会修改`r.ibd` 文件中的表空间id，这个id存在于每一个数据页中。所以如果是大文件每个数据页都要修改，`import` 执行需要一些时间，但跟前两种方法比起来还是非常快。

### 对比

1. 物理拷贝最快。
   - 必须全表拷贝
   - 必须到服务器上拷贝数据
   - 源表和目标表都是InnoDB引擎才能使用
2. mysqldump可以用where过滤导出部分数据，不能使用join这种比较复杂的where条件写法。可以跨引擎
3. `select ... into outfile` 最灵活，支持所有sql写法。每次只能导出一张表，表结构需要另外的语句单独备份。可以跨引擎。

## 42 grant之后要跟着flush privileges才能生效吗

`create user 'ua'@'%' identified by 'pa';`

创建用户'ua'@'%'，密码pa。mysql里user+host表示一个用户，ua@ip1和ua@ip2代表不同用户。

这条命令做了

1. 磁盘上，往mysql.user表插入一行，由于没指定权限，这行数据上所有表示权限的字段值都是N
2. 内存里往数组`acl_users` 里插入一个`acl_user` 对象，这个对象的`access` 字段值为0。

### 全局权限

全局权限作用于整个mysql实例，权限信息保存在mysql.user表里。

给用户ua赋予最高权限：

`grant all privileges on *.* to 'ua'@'%' with grant option;`

1. 磁盘上，mysql.user表里用户'ua'@'%'这一行所有表示权限的字段的值改为'Y'
2. 内存里，这个`acl_user` 对象的`access` 值修改为二进制的全1

`grant` 执行完后新的客户端使用ua登陆成功，mysql会为新连接维护一个线程对象，从`acl_users` 里查到这个用户的权限，将权限值拷贝到这个线程对象中。之后这个连接中关于全局权限的判断都直接用线程对象内部保存的权限值。这个信息不会被revole操作影响。

`grant` 命令完成后即时生效，之后新创建的连接会使用新权限；已经存在的连接的全局权限不受`grant` 影响。

回收上述`grant` 语句赋予的权限：

`revoke all privileges on *.* from 'ua'@'%';`

磁盘上和内存里的动作和`grant` 相反。

### db权限

让用户ua拥有库db1的所有权限：

`grant all privileges on db1.* to 'ua'@'%' with grant option;`

库的权限记录保存在mysql.db表，内存里保存在数组`acl_dbs` ，这条命令做了：

1. 磁盘上往mysql.db表中插入一行记录，所有权限字段设为'Y'
2. 内存里增加一个对象到`acl_dbs` 中，权限位为全1。

每次判断用户对数据库的权限时，需要遍历一次`acl_dbs` 数组，根据user、host、db找到对象，根据权限位判断。

但是如果某个会话已经处于某个db里（`use dbname;` ），执行use的时候拿到的库权限就会保存在会话变量中，也不受revoke影响，在切换出这个库之前，一直有use的时候拿到的权限。

以下例子是针对上述结论的说明

|      |                              A                               |                              B                               |                  C                  |
| :--: | :----------------------------------------------------------: | :----------------------------------------------------------: | :---------------------------------: |
|  T1  | `connnect(root,rootpwd);` <br /> `create database db1;` <br /> `create user 'ua'@'%' identified by 'pa';` <br /> `grant super on *.* to 'ua'@'%';` <br /> `grant all privileges on db1.* to 'ua'@'%';` |                                                              |                                     |
|  T2  |                                                              | `connect(ua,pa);` <br /> `set global sync_binlog=1;` (OK)<br /> `create table db1.t(c int);` (OK) | `connect(ua,pa);` <br /> `use db1;` |
|  T3  |             `revoke super on *.* from 'ua'@'%';`             |                                                              |                                     |
|  T4  |                                                              | `set global sync_binlog=1;` (OK)<br /> `alter table db1.t engine=innodb;` (OK) | `alter table t engine=innodb;` (OK) |
|  T5  |       `revoke all privileges on db1.* from 'ua'@'%';`        |                                                              |                                     |
|  T6  |                                                              | `set global sync_binlog=1;` (OK)<br /> `alter table db1.t engine=innodb;` (ALTER command denied) | `alter table t engine=innodb;` (OK) |

`set global sync_binlog` 需要super权限。

T3收回ua的super权限，T4执行`set global` 权限认证通过，是因为super是全局权限，权限信息存在会话的线程对象中，不受revoke影响。

T5去掉ua对db1库的所有权限，T6会话B权限不足，是因为revoke会修改内存里的`acl_dbs` 数组，即时生效，每次判断用户对数据库的权限，都会遍历一遍这个数组，所以revoke影响到B；而C通过`use db1;` 已经处于库db1中，use时拿到的权限保存在会话变量中（这里应该是服务端的变量？如果是存在客户端，那么只要客户端不删除这个信息，切换库之后还是可以通过服务端的权限认证），在C切换出db1之前，C对这个库就一直有权限。

### 表权限和列权限

表权限定义放在mysql.tables_priv表中，列权限定义放在mysql.columns_priv表中，两类权限组合起来放在内存的hash结构column_priv_hash中。

赋予权限：

```mysql
create table db1.t1(id int, a int);
 
grant all privileges on db1.t1 to 'ua'@'%' with grant option;
GRANT SELECT(id), INSERT (id,a) ON db1.t1 TO 'ua'@'%' with grant option;
```

这两个权限每次grant都会修改数据表和内存的hash结构，因此对这两类权限的操作会马上影响到已存在的连接。

### flush privileges使用场景

`flush privileges` 会清空`acl_users` 数组然后从`mysql.user` 中读取数据，重新构造一个`acl_users` ，即以表中数据为准，将全局权限内存数组重新加载。

对db、表、列权限mysql也做了同样处理。

grant/revoke会同步更新磁盘和内存，正常情况下grant之后没必要`flush privileges` 

往往是不规范的操作导致权限表的数据跟内存的权限数据不一致，才需要用这条语句。

不规范的操作比如用DML语句直接操作系统权限表，这会导致权限表磁盘数据跟内存中的权限信息不一致，可能导致被DML语句直接从系统表删除的用户

- 由于内存数组中还保有信息，仍然可以用这个用户连接数据库
- 给这个用户赋予权限失败，因为权限表中找不到这行；重新创建这个用户也不行，因为在内存中判断时，会认为用户还存在。

要删除用户，应该用drop语句，可以同步操作磁盘和内存。

## 43 是否使用分区表

**这篇文章针对的是单机上的单表多分区，而不是集群的分区表。** 

```mysql
CREATE TABLE t(
	ftime datetime NOT NULL,
	c int(11) DEFAULT NULL,
  KEY(ftime)
)ENGINE=InnoDB DEFAULT CHARSET=latin1
PARTITION BY RANGE(YEAR(ftime))
(PARTITION p_2017 VALUES LESS THAN (2017) ENGINE=InnoDB,
PARTITION p_2018 VALUES LESS THAN (2018) ENGINE=InnoDB,
PARTITION p_2019 VALUES LESS THAN (2019) ENGINE=InnoDB,
PARTITION p_others VALUES LESS THAN MAXVALUE ENGINE=InnoDB);

insert into t values('2017-4-1', 1),('2018-4-1', 1);
```

### 分区表是什么

**以下部分内容对mysql8版本不适用，8.0以上版本已经取消了frm文件，将表结构存在系统表空间中** 

原作者应该是5.x版本，在那个版本中，mysql的data目录下对应的库目录下，有以下文件：

![表t的磁盘文件](https://gitee.com/huang-yd/image_repo/raw/master/mysql45-lxb/43/1.png)

两行记录分别落在p_2018和p_2019两个分区上。

每个分区对应一个ibd文件。对引擎层来说，这是4个表，对server层来说，这是1个表。

#### 8.0.25版本的文件

表结构存在系统表空间中（ibdata，mysql.ibd），数据存放跟之前一样。以下是ls命令结果。

```bash
t#p#p_2017.ibd
t#p#p_2018.ibd
t#p#p_2019.ibd
t#p#p_others.ibd
```

### 分区表的引擎层行为

#### InnoDB

|      |                              A                               |                              B                               |
| :--: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  T1  | `begin;` <br /> `select * from t where ftime='2017-5-1' for update` |                                                              |
|  T2  |                                                              | `insert into t values('2018-2-1', 1);` (OK)<br /> `insert into t values('2017-12-1',1);` (blocked) |

如果是普通表，索引ftime上应该加间隙锁('2017-4-1', '2018-4-1')，B的两条语句应该都进入锁等待。

对于引擎来说，p_2018和p_2019是不同的表，即2017-4-1的下一个记录是p_2018分区的supremum，A的select只操作了分区p_2018，所以锁的范围是p_2018分区中索引ftime的间隙锁('2017-4-1', supremum)

#### MyISAM

**mysql8.0版本以上不允许创建myisam分区表。原作者的以下内容对5.x版本有效。** 



|                              A                               |                              B                               |
| :----------------------------------------------------------: | :----------------------------------------------------------: |
| `alter table t engine=myisam;` <br /> `update t set c=sleep(100) where ftime='2017-4-1'` |                                                              |
|                                                              | `select * from t where ftime='2018-4-1';` (OK)<br /> `select * from t where ftime='2017-5-1';` (blocked) |

对myisam引擎来说也是4个表，myisam只支持表锁，表锁在引擎层实现。A加的表锁，是锁在分区p_2018上，只会堵住其他会话在这个分区上执行的查询，落到其他分区的查询不受影响。

分区表和手工分表的区别主要在server层上。分区表打开表的行为广为诟病。

### 分区策略

**以下所有跟MyISAM有关的内容，在mysql8版本以上都不适用，因为从8版本开始取消MyISAM分区表。** 

每当第一次访问一个分区表时，mysql需要把所有分区都访问一遍，如果分区很多，可能会因为打开的文件个数超过`open_files_limit` 参数而报错。

如果是myisam引擎，在向分区表插入数据时可能会因为打开的文件过多而报错，但实际上只需要访问一个分区。

如果用InnoDB引擎，当引擎打开文件超过`innodb_open_files` 的值时会关掉一些之前打开的文件，因此即便分区个数大于`open_files_limit` InnoDB也不会报打开文件过多的错误。

myisam使用的分区策略称为通用分区策略，每次访问分区由server层控制。

mysql5.7.9InnoDB引入本地分区策略，在引擎内部管理打开分区的行为。

Mysql8.0开始只允许创建已经实现了本地分区策略的引擎的分区表，目前只有InnoDB和NDB两个引擎。

### 分区表在server层的行为

从server层看，一个分区表就只是一个表。

在加MDL锁时，访问一个分区会锁住整张表，导致其他会话操作无关的分区的DDL语句会被堵住。

分区表做DDL时影响会更大。如果用普通分表，DDL一个分表时，肯定不会跟另一个分表上的查询语句出现MDL锁冲突。

MDL锁之后的执行过程，引擎层认为分区表是不同的表，会根据分区表规则，只访问必要的分区，由where条件和分区规则判断。

如果查询语句的where条件没有分区的key，就只能访问所有分区，这跟正常的分表是一样的。

### 分区表的应用场景

方便清理历史数据。

`alter table t drop partition ...` 直接删除分区文件，效果跟drop普通表类似，比delete语句速度更快、对系统影响更小。

对业务透明。跟手动分表相比，业务代码更简洁。

**看下来好处微不足道。** 

## 44 答疑（三）

### Simple Nested Loop join和BNL的性能

BNL性能更好。虽然判断次数一样，但是每次SNLJ判断都要全表扫描被驱动表。

如果被驱动表数据没有在Buffer Pool，需要从磁盘读入，加入LRU链表中，影响其他业务命中率，并且因为多次全表扫描，更容易将被驱动表的数据页放到LRU链表的头部。

即使被驱动表的数据都在内存，LRU链表里也不一定全是被驱动表的数据页，（并且我认为每次都要遍历LRU链表，）且每次查找下一个记录的操作都类似指针操作，而`join_buffer` 中是数组，（并且只有驱动表的数据，没有无效的查找），遍历的成本更低。

### distinct 和 group by性能

假如表t的字段a上没有索引

```mysql
select a from t group by a order by null;
select distinct a from t;
```

**这不是标准的group by用法，不建议这么写** 

第一条语句的逻辑是按照字段a分组，相同的a值只返回一行。这就是distinct的语义。所以不执行group by的聚合函数时，group by和distinct的语义和执行流程相同，性能也相同。

执行流程：

1. 创建临时表，临时表有一个字段a，在字段a上创建一个唯一索引
2. 遍历表t，依次取数据插入临时表中，如果发现唯一键冲突就跳过，否则插入成功。
3. 遍历完成，将临时表作为结果集返回给客户端。

## 45 自增id用完怎么办

### 表定义自增值id

表定义的自增值达到上限后，再申请下一个id时，得到的值保持不变。因此可能造成主键冲突。

### InnoDB系统自增row_id

如果InnoDB表没有指定主键，InnoDB会创建一个不可见的6字节的`row_id` 做主键。InnoDB维护一个全局的`dict_sys.row_id` 值，所有无主键的InnoDB表，每插入一行，都将当前的这个值作为插入行的`row_id` ，然后把`dict_sys.row_id` 加1。

因此`row_id` 的范围是$$[0, 2^{48}-1]$$ 

当达到上限后，下一个值就是0，然后继续循环。

在InnoDB的逻辑里，申请到`row_id=N` 后，就将这行数据写入表中；如果表中已存在`row_id=N` 的行，原有的数据会被覆盖。

比起数据被覆盖丢失，表自增id到达上限后再插入报主键冲突错误，更能被接受。所以应该主动创建自增主键。

覆盖数据意味着数据丢失，影响数据可靠性。主键冲突是插入失败，影响可用性。一般情况下可靠性优先于可用性。

### Xid

是binlog和redo log用来匹配的字段，以检查redo log对应的binlog事务是否完整。

mysql内部维护全局变量`global_query_id` ，每次执行语句将它赋值给`Query_id` ，然后全局变量加1。如果当前语句是事务的第一条语句，同时把`Query_id` 赋值给事务的Xid。

`global_query_id` 是纯内存变量，重启清零。因此在同一个数据库实例中不同事务的Xid有可能相同。

mysql重启后会重新生成新的binlog文件，保证同一个文件里Xid唯一。

`global_query_id` 是8字节，达到上限后会溢出回到0。理论上讲同一个binlog里还是会出现相同的Xid，但是这要求在这一次mysql重启后执行$$2^{64}$$ 次查询，因此这个可能性可以忽略不计。

### InnoDB trx_id

Xid是用于日志的，由server层维护。InnoDB内部使用Xid是为了能够在InnoDB事务和server之间做关联。

而InnoDB自己的trx_id，是在事务隔离级别和事务可见性中用到的事务id（transaction id）

InnoDB维护一个全局变量`max_trx_id` ，每次需要申请`trx_id` 就获得它的当前值，然后`max_trx_id` 加1。

每一行数据都记录更新这行数据的事务的`trx_id` ，当一个事务读到一行数据时，通过事务的一致性视图与这行数据记录的`trx_id` 做对比来判断是否可见。

可以从`information_schema.innodb_trx` 表中看到正在执行的事务的`trx_id` 。

对于只读事务，InnoDB不分配`trx_id` ，此时通过`information_schema.innodb_trx` 表查出来的`trx_id` 值是由当前事务的`trx` 变量的指针地址转成整数再加上$$2^{48}$$ 得到。这个算法保证了

1. 同一个只读事务在执行期间，指针地址不会变，因此在`innodb_trx` 和`innodb_locks` 表里，同一个只读事务查出来的`trx_id` 一样。
2. 多个并行的只读事务，每个事务的`trx` 变量的指针地址不同，保证了不同的并发只读事务查出来的`trx_id` 不同。

在事务执行改动数据的语句时（包括`select...for update` 这些语义是写的语句），InnoDB才真正给事务分配`trx_id` 。

除了用户执行的语句，InnoDB内部的事务也会占用`trx_id` ，比如update和delete把数据放到purge队列等待物理删除、表的索引信息统计等。因此`trx_id` 并不是加1递增。

只读事务不分配`trx_id` 的好处：

- 减小事务视图里活跃事务数组的大小，创建一致性视图时，InnoDB只需要拷贝读写事务的`trx_id` 
- 减少`trx_id` 的申请次数，普通的查询语句不需要申请`trx_id` ，大大减少了并发事务申请`trx_id` 的锁冲突。

`max_trx_id` 会持久化，重启mysql也不会重置为0。跟`row_id` 一样是8字节，但上限是$$2^{48}-1$$ ，达到上限后从0开始。

达到上限时，事务的`trx_id` 是$$2^{48}-1$$ ，在可重复读的隔离级别下，也会出现脏读的bug，因为它的一致性视图低水位就是$$2^{48} - 1$$ ，在它之后执行的事务，`trx_id` 从0开始，小于低水位，因此对它可见。

### thread_id

`show processlist` 的第一列就是`thread_id` 。

系统保存一个4字节全局变量`thread_id_counter` ，每新建一个连接就将它的值赋给新连接的线程变量，达到$$2^{32}-1$$ 后重置为0。

`show processlist` 不会出现相同的`thread_id` ，因为mysql维护一个唯一数组，给新线程分配`thread_id` 时，往`thread_id` 数组插入这个`thread_id` 的值，如果数组里已经有这个值，就会阻塞。

