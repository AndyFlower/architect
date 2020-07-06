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
   GRANT REPLICATION SLAVE,REPLICATION CLIENT ON *.* TO'repl'@'192.168.8.147' IDENTIFIED BY '123456';
   FLUSH PRIVILEGES;
   ```

   

