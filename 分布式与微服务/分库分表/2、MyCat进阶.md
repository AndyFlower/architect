## MyCat进阶

> 目标：
>
> 1. 掌握不同数据分片策略的配置方式与特点
> 2. 理解Mycat注解的作用与应用场景
> 3. Mycat扩容与数据导入导出
> 4. Mycat核心原理分析总结

## 1.MySQL主从复制

### 1.1 主从复制的含义

在MySQL多服务器的架构中，至少要有一个主节点（master），跟主节点相对的，我们把它叫做从节点（slave）。主从复制，就是把主节点的数据复制到一个或者多个从节点。主服务器和从服务器可以在不同的IP上，通过远程连接来同步数据，这个是异步的过程。

### 1.2 主从复制的形式

- 一主一从/一主多从
- 多主一从
- 双主复制
- 级联复制

### 1.3 主从复制的用途

- 数据备份：把数据复制到不同的机器上，以免单台服务器发生故障时数据丢失。
- 读写分离：让主库负责写，从库负责读，从而提高读写的并发度。
- 高可用HA：当节点故障时，自动转移到其他节点，提高可用性。
- 扩展：结合负载的机制，均摊所有的应用访问请求，降低单机IO。

### 1.4 binlog

客户端对MySQL数据库进行操作的时候，包括DDL和DML语句，服务端会在日志文件中用事件的形式记录所有的操作记录，这个文件就是binlog文件（属于逻辑日志，跟Redis的AOF文件类似）。

基于binlog，我们可以实现主从复制和数据恢复。

Binlog默认是不开启的，需要在服务端手动配置。注意有一定的性能损耗.

#### 1.4.1 binlog配置

编辑/etc/my.cnf

```
lob-bin=/var/lib/mysql/mysql-bin
server-id=1
```

重启MySQL服务

```shell
service mysqld stop
service mysqld start
#如果出错查看日志
vi /var/log/mysqld.log
cd /var/lib/mysql
```

是否其binlog

```sql
 SHOW GLOBAL VARIABLES LIKE 'log_bin%'; 
```

#### 1.4.2 binlog格式

- STATEMENT：记录每一条修改数据的SQL语句（减少日志量，节约IO）。

- ROW：记录哪条数据被修改了，修改成什么样子了（5.7以后默认）。

- MIXED：结合两种方式，一般的语句用STATEMENT，函数之类的用ROW。

查看binlog格式：

```sql
 SHOW GLOBAL VARIABLES LIKE '%binlog_format%'; 
```

![binlog格式](images\binlog格式.png)

查看binlog列表：

```sql
SHOW BINARY LOGS;
```

查看binlog内容：

```sql
SHOW BINLOG EVENTS IN 'mysql-bin.000001';
```

### 1.5 主从复制原理

#### 1.5.1 主从复制配置

1. 主库开启binlog，设置server-id

2. 在主库创建具有复制权限的用户，允许从库连接

   ```sql
   GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO 'repl'@'192.168.8.147' IDENTIFIED BY '123456';
   FLUSH PRIVILEGES;
   ```

3. 从库/etc/my.inf配置，重启数据库

   ```
   server-id=2
   log-bin=mysql-bin
   relay-log=mysql-relay-bin
   read-only=1
   log-slave-updates=1
   ```

   log-slave-updates决定了在从binlog读取数据时，是否记录binlog,实现双主和级联的关键。

4. 在从库执行

   ```
   stop slave;
   change master to master_host='192.168.8.146',master_user='repl',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=4;
   start slave;
   ```

   

5. 查看同步状态

   ```
   SHOW SLAVE STATUS \G
   ```

   ![主从复制状态](images\主从复制状态.png)

#### 1.5.2 主从复制原理

这里面涉及到几个线程：

![主从复制原理](images\主从复制原理.png)

1. slave服务器执行start slave,开启主从复制开关，slave服务器的IO线程请求从master服务器读取binlog(如果该线程追赶上了主库，会进行睡眠状态)。
2. master服务器创建Log dump线程，把binlog发送给slave服务器，slave服务器把读取到的binlog日志内容写入中继日志relay log(会记录位置信息，以便下次继续读取)
3. slave服务器中的SQL线程会实时监测relay log中新增的日志内容，把relay log解析成SQL语句，并执行。

## 2.Mycat高可用

目前Mycat没有实现多对Mycat集群的支持，可以暂时使用HAProxy来做负载

思路：HAProxy对Mycat进行负载，Keepalived实现VIP

![Mycat高可用](images\Mycat高可用.png)

## 3.Mycat注解

### 3.1 注解的作用

当关联的数据不在同一个节点的时候，Mycat是无法实现跨库join的。

Mycat作为一个中间件，有很多自身不支持的SQL语句，比如存储过程，但是这些语句在实际的数据库节点上是可以执行的，有没有办法让Mycat做一层透明的代理转发，直接找到目标数据节点去执行这些SQL语句呢?

那我们必须要有一个方式告诉Mycat应该在哪个节点上执行。这个就是Mycat的注解。我们在需要执行的SQL语句前面加上一段代码，帮助Mycat找到我们的目标节点。

### 3.2 注解的用法

注解形式是：

```
/*!mycat:sql=注解SQL语句*/
```

注解 使用方式是

```
/*!mycat:sql=注解SQL语句*/真正执行的SQL
```

使用时将=后面的“注解执行语句”替换为需要的SQL语句即可。

使用注解有一些限制，或者注意的地方：

| 原始SQL | 注解SQL                                                      |
| ------- | ------------------------------------------------------------ |
| select  | 1. 选择能唯一确定分片的主表，如与用户表关联的时候可以选择用户表2. 若是业务需要在主表所在的各个分片上都执行可以不加能确定分片的条件 |
| insert  | 使用insert的表作为注解SQL，必须能确定到某个分片，原始SQL插入的字段必须包括分片字段。非分片表：必须能确定到某个分片。 |
| delete  | 1. 对于分片表使用要删除记录的表做注解 SQL                    |
| update  | 1. 对于分片表用所要更新的表做注解 SQL 2. 禁止更新分片表的分片列 3. 根据业务需要添加注解 Sql 的分片字段值 |
| Call    | 1. 若是要在所有的分片上都执行存储过程，则使用一个在所有分片上都包含的表，不添加任何分片条件 调用存储过程   2. 若是单个分片执行，使用能确认到这个分片的表以及分片条件 |

使用注解并不额外增加 MyCat 的执行时间；从解析复杂度以及性能考虑，注解 SQL 应尽量简单。至于一个SQL 使用注解和不使用注解的性能对比，不存在参考意义，因为前提是 MyCat 不支持的 SQL 才使用注解。

注解可以帮我们解决什么问题呢？

### 3.3 使用注解实例

#### 3.3.1 创建表或存储过程

customer.id=1全部路由到146

```sql
--存储过程
/*!mycat:sql=select * from customer where id=1*/ CREATE PROCEDURE test_proc BEGIN END;
--表
/*!mycat:sql=select * from customer where id=1*/ CREATE TABLE test2(id INT)
```

#### 3.3.2 特殊语句自定义分片

Mycat本身不支持insert select ,通过注解支持

```sql
/*!mycat :sql=select * from customer where id=1*/INSERT INTO test2(id) SELECT id FROM order_detail;
```

#### 3.3.3 多表ShareJoin

```
/*!mycat:catlet=io.mycat.catlets.ShareJoin*/select a.order_id,b.price from order_info a,order_detail b where a.nums=b.goods_id
```

读写分离

读写分离:配置Mycat读写分离后，默认查询都会从读节点获取数据，但是有些场景需要获取实时数据，如果从读节点获取数据可能因延时而无法实现实时，Mycat支持通过注解`/*balance*/`来强制从写节点（writehost）查询数据.

```sql
/*balance*/ select a.* from customer a where a.id=888;
```

#### 3.3.4 读写分离数据库选择（1.6版本之后）

```
/*!mycat:db_type=master*/select * from customer;
/*!mycat:db_type=master*/select * from customer;
/*#mycat:db_type=master*/select * from customer;
/*#mycat:db_type=master*/select * from customer;
```

注解支持的'!'不被 mysql 单库兼容，

注解支持的'#'不被 mybatis 兼容

### 3.4 注解原理

MyCat 执行 SQL 语句的流程是先进行 SQL 解析处理，解析出分片信息(路由信息)后，然后到该分片对应的物理库上去执行；若传入的 SQL 语句 MyCat 无法解析，则 MyCat 不会去执行；而注解则是告诉 MyCat 按照注解内的 SQL（称之为注解 SQL）去进行解析处理，解析出分片信息后，将注解后真正要执行的 SQL 语句（称之为原始 SQL）发送到该分片对应的物理库上去执行。

从上面的原理可以看到，注解只是告诉 MyCat 到何处去执行原始 SQL；因而使用注解前，要清楚的知道该原始 SQL 去哪个分片执行，然后在注解 SQL 中也指向该分片，这样才能使用！例子中的 sharding_id=10010 即是指明分片信息的。

需要说明的是，若注解 SQL 没有能明确到具体某个分片，譬如例子中的注解 SQL 没有添加

sharding_id=10010 这个条件，则 MyCat 会将原始 SQL 发送到 persons 表所在的所有分片上去执行去，这样造成的后果若是插入语句，则在多个分片上都存在重复记录，同样查询、更新、删除操作也会得到错误的结果！

## 4.分片策略详解

分片的目标是将大量数据和访问请求均匀分布在多个节点上，通过这种方式提升数据服务的存储和负载能力。

### 4.1 Mycat分片策略详解

总体上分为连续分片和离散分片，还有一种是连续分片和离散分片的结合，例如先范围后取模。

比如范围分片（id或者时间）就是典型的连续分片，单个分区的数量和边界是确定的。离散分片的分区总数量和边界是确定的，例如对key进行哈希运算，或者再取模。

关键词：范围查询、热点数据、扩容

连续分片优点：

1. 范围条件查询消耗资源少（不需要汇总数据）
2. 扩容无需迁移数据（分片固定）

连续分片缺点：

1. 存在数据热点的可能性

2. 并发访问能力受限于单一或少量DataNode（访问集中）


离散分片优点：

1. 并发访问能力增强（负载到不同的节点）

2. 范围条件查询性能提升（并行计算）


离散分片缺点：1）数据扩容比较困难，涉及到数据迁移问题

