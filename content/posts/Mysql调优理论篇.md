---
author: ["柿子"]
title: "Mysql调优理论篇"
date: "2021-08-17 19:59:25"
description: "MySQL调优理论."
summary: "关于MySQL调优理论."
tags: ["Mysql", "调优"]
---

## 性能监控工具

### 服务端的配置和性能

#### show profile

```sql
# 案例
# all：显示所有性能信息
show profile all for query n

# block io：显示块io操作的次数
show  profile block io for query n

# context switches：显示上下文切换次数，被动和主动
show profile context switches for query n

# cpu：显示用户cpu时间、系统cpu时间
show profile cpu for query n

# IPC：显示发送和接受的消息数量
show profile ipc for query n

# page faults：显示页错误数量
show profile page faults for query n

# source：显示源码中的函数名称与位置
show profile source for query n

# swaps：显示swap的次数
show profile swaps for query n
```

### 运行时性能

#### performance_schema

##### 简介

- 它是数据库中的库，使用的存储引擎是performance_schema，主要关注的是数据库运行过程中的性能相关的数据，与`information_schema`不同，关注的是数据库表的元数据
- server内部在发生`函数调用`、`操作系统的等待`、`SQL语句的执行阶段`、`单个SQL或者多个SQL的集合`的事件来触发采集`消耗`、`耗时`、`活动执行次数`等信息，并且事件的采集可以方便的提供server中的相关存储引擎对`磁盘文件`、`表I/O`、`表锁`等资源同步多用相关信息
- 它的记录只会在server本地，不会记录到binlog，同时也不复制到其他的server，其实在mysql源代码实现过程中主要是通过检查点（埋点）的方式收集
- 可以通过select查询，同时也还可以修改相关收集配置，动态修改`setup_*`开头的几个配置表

##### 表分类

```sql
-- 事件表：当前语句、历史语句、长语句历史、聚合后的摘要summary
-- 其中，summary表还可以根据帐号(account),主机(host),程序(program),
-- 线程(thread),用户(user)和全局(global)再进行细分)
show tables like '%statement%';

-- 等待事件记录表，与语句事件类型的相关记录表类似：
show tables like '%wait%';

-- 阶段事件记录表，记录语句执行的阶段事件的表
show tables like '%stage%';

-- 事务事件记录表，记录事务相关的事件的表
show tables like '%transaction%';

-- 监控文件系统层调用的表
show tables like '%file%';

-- 监视内存使用的表
show tables like '%memory%';

-- 动态对performance_schema进行配置的配置表
show tables like '%setup%';
```

##### 入门使用

首先需要查看是否开启此功能，需要显示的修改[my.cnf]配置文件

```sql
-- 查看performance_schema的属性
mysql> SHOW VARIABLES LIKE 'performance_schema';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| performance_schema | ON    |
+--------------------+-------+
1 row in set (0.01 sec)

-- 在配置文件中修改performance_schema的属性值，on表示开启，off表示关闭
[mysqld]
performance_schema=ON

-- 切换数据库
use performance_schema;

-- 查看当前数据库下的所有表,会看到有很多表存储着相关的信息
show tables;

-- 可以通过show create table tablename来查看创建表的时候的表结构
mysql> show create table setup_consumers;
+-----------------+---------------------------------
| Table           | Create Table                    
+-----------------+---------------------------------
| setup_consumers | CREATE TABLE `setup_consumers` (
  `NAME` varchar(64) NOT NULL,                      
  `ENABLED` enum('YES','NO') NOT NULL               
) ENGINE=PERFORMANCE_SCHEMA DEFAULT CHARSET=utf8 |  
+-----------------+---------------------------------
1 row in set (0.00 sec) 
```

##### 两个概念

- instruments：生产者，用于采集mysql中各种各样的操作产生的事件信息，对应配置表中的配置项我们可以称为监控采集配置项，前面动态表`setup_*配置`
- consumers：消费者，对应的消费者表用于存储来自instruments采集的数据，对应配置表中的配置项我们可以称为消费存储配置项

##### 常用配置项

###### 启动配置

```shell
# 是否在mysql server启动时就开启events_statements_current表的记录功能(该表记录当前的语句事件信息)，
# 启动之后也可以在setup_consumers表中使用UPDATE语句进行动态更新setup_consumers配置
# 表中的events_statements_current配置项，默认值为TRUE
performance_schema_consumer_events_statements_current=TRUE

# 与performance_schema_consumer_events_statements_current选项类似，
# 但该选项是用于配置是否记录语句事件短历史信息，默认为TRUE
performance_schema_consumer_events_statements_history=TRUE

# 与performance_schema_consumer_events_statements_current选项类似，
# 但该选项是用于配置是否记录语句事件长历史信息，默认为FALSE
performance_schema_consumer_events_stages_history_long=FALSE

# 除了statement(语句)事件之外，还支持：wait(等待)事件、state(阶段)事件、transaction(事务)事件，
# 他们与statement事件一样都有三个启动项分别进行配置，但这些等待事件默认未启用，
# 如果需要在MySQL Server启动时一同启动，则通常需要写进my.cnf配置文件中

# 是否在MySQL Server启动时就开启全局表（如：
# mutex_instances、rwlock_instances、cond_instances、file_instances、
# users、hostsaccounts、socket_summary_by_event_name、file_summary_by_instance
# 等大部分的全局对象计数统计和事件汇总统计信息表 ）的记录功能，
# 启动之后也可以在setup_consumers表中使用UPDATE语句进行动态更新全局配置项,默认值为TRUE
performance_schema_consumer_global_instrumentation=TRUE

# 是否在MySQL Server启动时就开启events_statements_summary_by_digest 表的记录功能，
# 启动之后也可以在setup_consumers表中使用UPDATE语句进行动态更新digest配置项,默认值为TRUE
performance_schema_consumer_statements_digest=TRUE

# 是否在MySQL Server启动时就开启
performance_schema_consumer_thread_instrumentation=TRUE

# events_xxx_summary_by_yyy_by_event_name表的记录功能，
# 启动之后也可以在setup_consumers表中使用UPDATE语句进行动态更新线程配置项,默认值为TRUE

performance_schema_instrument[=name]
# 是否在MySQL Server启动时就启用某些采集器，由于instruments配置项多达数千个，
# 所以该配置项支持key-value模式，还支持%号进行通配等，如下:
# [=name]可以指定为具体的Instruments名称（但是这样如果有多个需要指定的时候，就需要使用该选项多次），
# 也可以使用通配符，可以指定instruments相同的前缀+通配符，
# 也可以使用%代表所有的instruments指定开启单个instruments
performance-schema-instrument='instrument_name=value'

# 使用通配符指定开启多个instruments
performance-schema-instrument='wait/synch/cond/%=COUNTED'

# 开关所有的instruments
performance-schema-instrument='%=ON'
performance-schema-instrument='%=OFF'

# 注意，这些启动选项要生效的前提是，需要设置performance_schema=ON。
# 另外，这些启动选项虽然无法使用show variables语句查看，
# 但我们可以通过setup_instruments和setup_consumers表查询这些选项指定的值。
```

###### 系统变量

```shell
# 查询是否开启运行时事件记录功能
show variables like '%performance_schema%';
# 重要的属性解释
# 控制performance_schema功能的开关，要使用MySQL的performance_schema，需要在mysqld启动时启用，以启用事件收集功能，该参数
# 在5.7.x之前支持performance_schema的版本中默认关闭，5.7.x版本开始默认开启
# 注意：如果mysqld在初始化performance_schema时发现无法分配任何相关的内部缓冲区，则performance_schema将自动禁用，并将performance_schema设置为OFF
performance_schema=ON

# 控制events_statements_summary_by_digest表中的最大行数。如果产生的语句摘要信息超过此最大值，
# 便无法继续存入该表，此时performance_schema会增加状态变量
performance_schema_digests_size=10000

# 控制events_statements_history_long表中的最大行数，
# 该参数控制所有会话在events_statements_history_long表中能够存放的
# 总事件记录数，超过这个限制之后，最早的记录将被覆盖全局变量，
# 只读变量，整型值，5.6.3版本引入 * 5.6.x版本中，5.6.5及其之前的版
# 本默认为10000，5.6.6及其之后的版本默认值为-1，通常情况下，
# 自动计算的值都是10000 * 5.7.x版本中，默认值为-1，通常情况下，自动
# 计算的值都是10000
performance_schema_events_statements_history_long_size=10000

# 控制events_statements_history表中单个线程（会话）的最大行数，该参数控制单个会话在
# events_statements_history表中能够存放的事件记录数，超过这个限制之后，单个会话最早的记录将被覆盖
# 全局变量，只读变量，整型值，5.6.3版本引入 * 5.6.x版本中，5.6.5及其之前的版本默认为10，5.6.6及其之后的版本默认值为-1，
# 通常情况下，自动计算的值都是10 * 5.7.x版本中，默认值为-1，通常情况下，自动计算的值都是10
# 除了statement(语句)事件之外，wait(等待)事件、state(阶段)事件、transaction(事务)事件，
# 他们与statement事件一样都有三个参数分别进行存储限制配置，有兴趣的同学自行研究，这里不再赘述
performance_schema_events_statements_history_size=10

# 用于控制标准化形式的SQL语句文本在存入performance_schema时的限制长度，
# 该变量与max_digest_length变量相关(max_digest_length变量含义请自行查阅相关资料)全局变量，
# 只读变量，默认值1024字节，整型值，取值范围0~1048576
performance_schema_max_digest_length=1024

# 控制存入events_statements_current，events_statements_history和events_statements_history_long语句事件表中的
# SQL_TEXT列的最大SQL长度字节数。 超出系统变量performance_schema_max_sql_text_length的部分将被丢弃，不会记录，一般情况
# 下不需要调整该参数，除非被截断的部分与其他SQL比起来有很大差异
# 全局变量，只读变量，整型值，默认值为1024字节，取值范围为0~1048576，5.7.6版本引入
# 降低系统变量performance_schema_max_sql_text_length值可以减少内存使用，但如果汇总的SQL中，被截断部分有较大差异，会导致没
# 有办法再对这些有较大差异的SQL进行区分。 增加该系统变量值会增加内存使用，但对于汇总SQL来讲可以更精准地区分不同的部分。
performance_schema_max_sql_text_length=1024
```

##### setup_*配置表说明

```sql
/*
performance_timers表中记录了server中有哪些可用的事件计时器
字段解释：
	timer_name:表示可用计时器名称，CYCLE是基于CPU周期计数器的定时器
	timer_frequency:表示每秒钟对应的计时器单位的数量,CYCLE计时器的换算值与CPU的频率相关、
	timer_resolution:计时器精度值，表示在每个计时器被调用时额外增加的值
	timer_overhead:表示在使用定时器获取事件时开销的最小周期值
*/
select * from performance_timers;

/*
setup_timers表中记录当前使用的事件计时器信息
字段解释：
	name:计时器类型，对应某个事件类别
	timer_name:计时器类型名称
*/
select * from setup_timers;

/*
setup_consumers表中列出了consumers可配置列表项
字段解释：
	NAME：consumers配置名称
	ENABLED：consumers是否启用，有效值为YES或NO，此列可以使用UPDATE语句修改。
*/
select * from setup_consumers;

/*
setup_instruments 表列出了instruments 列表配置项，即代表了哪些事件支持被收集：
字段解释：
	NAME：instruments名称，instruments名称可能具有多个部分并形成层次结构
	ENABLED：instrumetns是否启用，有效值为YES或NO，此列可以使用UPDATE语句修改。如果设置为NO，则这个instruments不会被执行，
	不会产生任何的事件信息
	TIMED：instruments是否收集时间信息，有效值为YES或NO，此列可以使用UPDATE语句修改，如果设置为NO，则这个instruments不会收
	集时间信息
*/
SELECT * FROM setup_instruments;

/*
setup_actors表的初始内容是匹配任何用户和主机，因此对于所有前台线程，默认情况下启用监视和历史事件收集功能
字段解释：
	HOST：与grant语句类似的主机名，一个具体的字符串名字，或使用“％”表示“任何主机”
	USER：一个具体的字符串名称，或使用“％”表示“任何用户”
	ROLE：当前未使用，MySQL 8.0中才启用角色功能
	ENABLED：是否启用与HOST，USER，ROLE匹配的前台线程的监控功能，有效值为：YES或NO
	HISTORY：是否启用与HOST， USER，ROLE匹配的前台线程的历史事件记录功能，有效值为：YES或NO
*/
SELECT * FROM setup_actors;

/*
setup_objects表控制performance_schema是否监视特定对象。默认情况下，此表的最大行数为100行。
字段解释：
	OBJECT_TYPE：instruments类型，有效值为：“EVENT”（事件调度器事件）、“FUNCTION”（存储函数）、“PROCEDURE”（存储过程）、
	“TABLE”（基表）、“TRIGGER”（触发器），TABLE对象类型的配置会影响表I/O事件（wait/io/table/sql/handler instrument）和
	表锁事件（wait/lock/table/sql/handler instrument）的收集
	OBJECT_SCHEMA：某个监视类型对象涵盖的数据库名称，一个字符串名称，或“％”(表示“任何数据库”)
	OBJECT_NAME：某个监视类型对象涵盖的表名，一个字符串名称，或“％”(表示“任何数据库内的对象”)
	ENABLED：是否开启对某个类型对象的监视功能，有效值为：YES或NO。此列可以修改
	TIMED：是否开启对某个类型对象的时间收集功能，有效值为：YES或NO，此列可以修改
*/
SELECT * FROM setup_objects;

/*
threads表对于每个server线程生成一行包含线程相关的信息，
字段解释：
	THREAD_ID：线程的唯一标识符（ID）
	NAME：与server中的线程检测代码相关联的名称(注意，这里不是instruments名称)
	TYPE：线程类型，有效值为：FOREGROUND、BACKGROUND。分别表示前台线程和后台线程
	PROCESSLIST_ID：对应INFORMATION_SCHEMA.PROCESSLIST表中的ID列。
	PROCESSLIST_USER：与前台线程相关联的用户名，对于后台线程为NULL。
	PROCESSLIST_HOST：与前台线程关联的客户端的主机名，对于后台线程为NULL。
	PROCESSLIST_DB：线程的默认数据库，如果没有，则为NULL。
	PROCESSLIST_COMMAND：对于前台线程，该值代表着当前客户端正在执行的command类型，如果是sleep则表示当前会话处于空闲状态
	PROCESSLIST_TIME：当前线程已处于当前线程状态的持续时间（秒）
	PROCESSLIST_STATE：表示线程正在做什么事情。
	PROCESSLIST_INFO：线程正在执行的语句，如果没有执行任何语句，则为NULL。
	PARENT_THREAD_ID：如果这个线程是一个子线程（由另一个线程生成），那么该字段显示其父线程ID
	ROLE：暂未使用
	INSTRUMENTED：线程执行的事件是否被检测。有效值：YES、NO 
	HISTORY：是否记录线程的历史事件。有效值：YES、NO * 
	THREAD_OS_ID：由操作系统层定义的线程或任务标识符（ID）：
*/
select * from threads
```

##### 实践案例

了解相关的参数配置后，可以对表进行一些实际分析，

```sql
-- 1、哪类的SQL执行最多？
SELECT DIGEST_TEXT,COUNT_STAR,FIRST_SEEN,LAST_SEEN FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC

-- 2、哪类SQL的平均响应时间最多？
SELECT DIGEST_TEXT,AVG_TIMER_WAIT FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC

-- 3、哪类SQL排序记录数最多？
SELECT DIGEST_TEXT,SUM_SORT_ROWS FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC

-- 4、哪类SQL扫描记录数最多？
SELECT DIGEST_TEXT,SUM_ROWS_EXAMINED FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC

-- 5、哪类SQL使用临时表最多？
SELECT DIGEST_TEXT,SUM_CREATED_TMP_TABLES,SUM_CREATED_TMP_DISK_TABLES FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC

-- 6、哪类SQL返回结果集最多？
SELECT DIGEST_TEXT,SUM_ROWS_SENT FROM events_statements_summary_by_digest ORDER BY COUNT_STAR DESC

-- 7、哪个表物理IO最多？
SELECT file_name,event_name,SUM_NUMBER_OF_BYTES_READ,SUM_NUMBER_OF_BYTES_WRITE FROM file_summary_by_instance ORDER BY SUM_NUMBER_OF_BYTES_READ + SUM_NUMBER_OF_BYTES_WRITE DESC

-- 8、哪个表逻辑IO最多？
SELECT object_name,COUNT_READ,COUNT_WRITE,COUNT_FETCH,SUM_TIMER_WAIT FROM table_io_waits_summary_by_table ORDER BY sum_timer_wait DESC

-- 9、哪个索引访问最多？
SELECT OBJECT_NAME,INDEX_NAME,COUNT_FETCH,COUNT_INSERT,COUNT_UPDATE,COUNT_DELETE FROM table_io_waits_summary_by_index_usage ORDER BY SUM_TIMER_WAIT DESC

-- 10、哪个索引从来没有用过？
SELECT OBJECT_SCHEMA,OBJECT_NAME,INDEX_NAME FROM table_io_waits_summary_by_index_usage WHERE INDEX_NAME IS NOT NULL AND COUNT_STAR = 0 AND OBJECT_SCHEMA <> 'mysql' ORDER BY OBJECT_SCHEMA,OBJECT_NAME;

-- 11、哪个等待事件消耗时间最多？
SELECT EVENT_NAME,COUNT_STAR,SUM_TIMER_WAIT,AVG_TIMER_WAIT FROM events_waits_summary_global_by_event_name WHERE event_name != 'idle' ORDER BY SUM_TIMER_WAIT DESC

-- 12-1、剖析某条SQL的执行情况，包括statement信息，stege信息，wait信息
SELECT EVENT_ID,sql_text FROM events_statements_history WHERE sql_text LIKE '%count(*)%';

-- 12-2、查看每个阶段的时间消耗
SELECT event_id,EVENT_NAME,SOURCE,TIMER_END - TIMER_START FROM events_stages_history_long WHERE NESTING_EVENT_ID = 1553;

-- 12-3、查看每个阶段的锁等待情况
SELECT event_id,event_name,source,timer_wait,object_name,index_name,operation,nesting_event_id FROM events_waits_history_longWHERE nesting_event_id = 1553;
```

[参考官方performance_schema教程](https://dev.mysql.com/doc/refman/5.7/en/performance-schema.html)

### show processlist

当前由服务器内执行的线程集执行的操作情况，更多可以参考官网：[Sources of Process Information](https://dev.mysql.com/doc/refman/5.7/en/processlist-access.html#processlist-sources)

## Sechema与数据类型优化

### 数据类型优化

- 三原则：占用越小越好，足够简单，避免为null

- 实际操作建议

  - 整型：TINYINT，SMALLEST，MEDIUMINT，INT，BIGINT分别使用8，16，24，32，64位存储空间

  - 字符和字符串：char（255字）、vachar（65535字）、text类（TINYTEXT:2^8^-1、TEXT:2^16^ -1、MEDIUMTEXT:2^24^-1、LONGTEXT:4G或2^32^-1）单位字节

  - BLOB和TEXT类型，分别是二进制和字符串格式来存储

  - datatime和timestamp

    - datatime（8字节）
      - 与时区无关，数据库底层对datetime无效
      - 可以精确到毫秒
      - 可保存的范围大
      - `字符串存储会导致确实时间的精度`
    - date（3字节）
      - 占用的字节数比字符串、datatime、int少
      - 可以通过date类型进行日期计算
      - 保存范围是1000-01-01到9999-12-31
    - timestamp（4字节）
      - 时间范围是linux元年1970-01-01开始到2038-01-19
      - 精确到秒
      - 采用整型存储
      - 依赖数据库的时区
      - 自动更新timestamp的列值

  - 尝试使用枚举代替字符串，mysql存储枚举类型非常的紧凑，有利于提升IO读取

  - 存储特殊类型通过可以通过函数转换如IP存储

     ```sql
     	# 序列化
      mysql> select inet_aton('1.1.1.1');
      +----------------------+
      | inet_aton('1.1.1.1') |
      +----------------------+
      |             16843009 |
      +----------------------+
      1 row in set (0.00 sec)
      
      # 反序列化
      mysql> select inet_ntoa(16843009);
      +---------------------+
      | inet_ntoa(16843009) |
      +---------------------+
      | 1.1.1.1             |
      +---------------------+
      1 row in set (0.00 sec)
     ```

### 合理的范式和反范式

- 合理的范式
  - 优点：更新更快，基本不会出现重复的数据，在内存中操作比较快
  - 缺点：需要不断的关联，增加IO消耗
- 反范式
  - 优点：所有的信息都在一张表中，可以避免关联，减少IO消耗
  - 缺点：冗余较多，删除操作时会删除不必要信息
- 实践
  - 项目前：先范式设计后分析业务反范式冗余存储，更新优先级根据业务来看（及时和延迟）
  - 项目中：分析业务代码逻辑，是否需要优化数据表增加冗余字段减少IO的开销

### 主键选择

- 代理主键
  - 与业务无关，无意义的数字序列，如ID

- 自然主键
  - 和业务相关，事物属性唯一标识，如身份证

- 如何选择
  - 代理主键更好，不与业务耦合，更加容易维护，统一的策略，也减少源代码数量

### 字符集选择

mysql精确到字段取优化，合适的字符集，一定程度上也会减少IO

- latin*：纯拉丁内容适合
- utf-8*：多语言内容

### 存储引擎选择

![image-20210817232400702](https://i.loli.net/2021/08/19/YcgjQklxv6XGoMq.png)

### 适当的拆分

- 比如有字段类型为TEXT之类大型字段，可以尝试拆到单独的表，当查询不需要此字段的查询是，可以大大减少IO的处理时间

## 执行计划EXPLAIN

### 格式：explain + SQL

### EXPLAIN字段解析

```sql
mysql> explain select inet_aton('1.1.1.1')\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: NULL
   partitions: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: No tables used
1 row in set, 1 warning (0.00 sec)
```

#### id有几种情况

1. id相同，顺序执行
2. id不同，递增，id号越大，越先执行
3. id相同和id不同同时存在，即第一种和第二种结合
4. null，代表是中间结果集（在8.x有些情况被优化了，5.x中还是存在的，比如 `UNION RESULT`）

#### select_type

```sql
-- sample:简单的查询，不包含子查询和union
explain select * from emp;

-- primary:查询中若包含任何复杂的子查询，最外层查询则被标记为Primary
explain select staname,ename supname from (select ename staname,mgr from emp) t join emp on t.mgr=emp.empno ;

-- union:若第二个select出现在union之后，则被标记为union
explain select * from emp where deptno = 10 union select * from emp where sal >2000;

-- dependent union:跟union类似，此处的depentent表示union或union all联合而成的结果会受外部表影响
explain select * from emp e where e.empno  in ( select empno from emp where deptno = 10 union select empno from emp where sal >2000)

-- union result:从union表获取结果的select
explain select * from emp where deptno = 10 union select * from emp where sal >2000;

-- subquery:在select或者where列表中包含子查询
explain select * from emp where sal > (select avg(sal) from emp) ;

-- dependent subquery:subquery的子查询要受到外部表查询的影响
explain select * from emp e where e.deptno in (select distinct deptno from dept);

-- DERIVED: from子句中出现的子查询，也叫做派生类，
explain select staname,ename supname from (select ename staname,mgr from emp) t join emp on t.mgr=emp.empno ;

-- UNCACHEABLE SUBQUERY：表示使用子查询的结果不能被缓存
 explain select * from emp where empno = (select empno from emp where deptno=@@sort_buffer_size);
 
-- uncacheable union:表示union的查询结果不能被缓存：sql语句未验证
```

#### table

1. 如果是具体的表名，则表明从实际的物理表中获取数据，当然也可以是表的别名
2. 表名是derivedN的形式，表示使用了id为N的查询产生的衍生表
3. 当有union result的时候，表名是union n1,n2等的形式，n1,n2表示参与union的id

#### type

`system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL `

```sql
-- all:全表扫描，一般情况下出现这样的sql语句而且数据量比较大的话那么就需要进行优化。
explain select * from emp;

-- index：全索引扫描这个比all的效率要好，主要有两种情况，一种是当前的查询时覆盖索引，
-- 即我们需要的数据在索引中就可以索取，或者是使用了索引进行排序，这样就避免数据的重排序
explain  select empno from emp;

-- range：表示利用索引查询的时候限制了范围，在指定范围内进行查询，
-- 这样避免了index的全索引扫描，适用的操作符： =, <>, >, >=, <, <=, IS NULL, BETWEEN, LIKE, or IN() 
explain select * from emp where empno between 7000 and 7500;

-- index_subquery：利用索引来关联子查询，不再扫描全表
explain select * from emp where emp.job in (select job from t_job);

-- unique_subquery:该连接类型类似与index_subquery,使用的是唯一索引
explain select * from emp e where e.deptno in (select distinct deptno from dept);
 
-- index_merge：在查询过程中需要多个索引组合使用，没有模拟出来

-- ref_or_null：对于某个字段即需要关联条件，也需要null值的情况下，查询优化器会选择这种访问方式
explain select * from emp e where  e.mgr is null or e.mgr=7369;

-- fulltext：全文索引

-- ref：使用了非唯一性索引进行数据的查找
create index idx_3 on emp(deptno);
explain select * from emp e,dept d where e.deptno =d.deptno;

-- eq_ref：使用唯一性索引进行数据查找
explain select * from emp,emp2 where emp.empno = emp2.empno;

-- const：这个表至多有一个匹配行，
explain select * from emp where empno = 7369;
 
-- system：表只有一行记录（等于系统表），这是const类型的特例，平时不会出现
```

#### possible_keys

这张table可能用的索引，不一定使用了

#### key

实际用到的索引，如果为null，则没有用索引，如果使用了覆盖索引，则会与select重叠

#### key_len

表示使用索引的字节长度，不损失精度情况，越短越好

#### ref

显示哪一列被使用了索引，是一个常数

#### rows

根据表的统计信息及索引情况，大致的计算需要读取的行数

#### extra

```sql
-- using filesort:说明mysql无法利用索引进行排序，只能利用排序算法进行排序，会消耗额外的位置
explain select * from emp order by sal;

-- using temporary:建立临时表来保存中间结果，查询完成之后把临时表删除
explain select ename,count(*) from emp where deptno = 10 group by ename;

-- using index:这个表示当前的查询时覆盖索引的，直接从索引中读取数据，
-- 而不用访问数据表。如果同时出现using where 表名索引被用来执行索引键值的查找，
-- 如果没有，表面索引被用来读取数据，而不是真的查找
explain select deptno,count(*) from emp group by deptno limit 10;

-- using where:使用where进行条件过滤
explain select * from t_user where id = 1;

-- using join buffer:使用连接缓存，情况没有模拟出来

-- impossible where：where语句的结果总是false
explain select * from emp where empno = 7469;
```

### 如何应用

- type判断是否有使用索引，尽可能的到达range<type<ref
- ref判断是否用到索引
- extra判断是否覆盖索引、排序是否用到索引

## 索引优化

### 索引基础

比如一本书的目录，可以让你快速的找到你感兴趣的内容

#### 优点

- 减少服务器扫描数据量
- 帮助服务器避免排序和临时表
- 将随机IO变成顺序IO

#### 用处

- 快速的找到WHERE字句的行
- 优化器会找到最优索引
- 如果表具有多列索引，优化器会使用索引的任何最左前缀来找到行
- 当有表连接时，，从其他表检索行数据
- 找到特定的索引列min和max值
- 如果索引和分组时在可用的索引最左前缀上完成的，则对进行排序和分组
- 某些情况下，可以查询检索值，无需检索行，如**覆盖索引**的情况

#### 分类

- 主键索引：PRIMARY KEY
- 唯一索引：UNIQUE
- 普通索引：NORMAL
- 全文索引：FULLTEXT
- 组合索引：KEY `name`(`字段1`,`字段2`,`字段3`)

#### 一些名词

- 回表：比如我们需要查用户信息，通过name去查（name字段列是普通索引），`普通索引`中的data存储的是`主键索ID`,`主键索引`data存放的是`数据行`，执行器会通过普通索引定位到data后，再拿着ID去`主键索引`去查用户信息数据行(**最后去主键索引查数据行的行为叫回表**)，遍历2次B+树
- 覆盖索引：还是刚刚的例子，如果查询的不是用户信息，而是用户ID（主键ID），去掉上面例子中去主键索引的过程，就是覆盖索引，**因为普通索引data就是用户ID**，遍历1次B+树
- 最左匹配：B+树索引是有序的，从左往右递增，而组合索引就是从表的字段的左边字段开始匹配，遇到范围停止匹配
- 索引下推：将where后面的条件需要在server层过滤的变为在server层之前过滤完成，交给server层的就是结果集

#### 索引常用的数据结构

- 哈希表+链表
- B+树
  - 发展历程
    - =>哈希表，查询时间复杂度（O(N)），为了降低时间复杂度人们想到了（logn）-> 二叉树
    - =>二叉树（分支倾斜严重）
    - =>平衡二叉树（平衡分支耗时）
    - =>红黑树（减少平衡距离，降低插入速度）
    - ---分割线---无论怎么优化二叉树，随着时间的推移，树的深度总会越来越深---分割线---
    - =>B树：
      - 每个节点（主键（主键+data）+指针（主键子节点范围））16K
      - 这样的缺点，节点容量固定条件下，data越大，能存放的指针变小，从而导致索引容纳的数少
      - ![image-20210818231739010](https://i.loli.net/2021/08/19/LSwECruVYlIcgXO.png)
    - =>B+树
      - 让最终的子节点去存放data，子节点的父节点都是指针
      - 最底层data之间还通过链表相互连接，方便遍历
      - ![image-20210818231629576](https://i.loli.net/2021/08/19/Em38aGv2XuHoZ9c.png)

#### 索引匹配的方式

name，age组合索引

- 全值匹配：name=‘张三’
- 匹配最左前缀：name=‘张三’ AND age=10
- 匹配列前缀：name like ‘张%’
- 匹配范围值：age>10
- 精确匹配某一列+范围匹配：name=‘’ AND age > 10
- 只访问索引列：select name,age from user where name=‘张三’ AND age=10(覆盖索引)

### 哈希索引

- 结构：哈希表 + 链表
- 优点
  - 结构非常的紧凑，然后查询很快
- 缺点
  - 哈希索引结构只包含行指针和哈希值，不存储值
  - 哈希不是顺序存储，无法排序
  - 不支持分列匹配查找
  - 只支持等值 =，or查找，不支持范围查找
  - 冲突严重时，链表特别长耗时，维护耗时
- 案例
  - 比如需要存放一个很长的URL，需求需要通过URL查询
  - 我们可以通过一些哈希函数，如CRC32
  - 这样可以减小体积

### 组合索引

- 多列共同组成索引树，where最左使用

### 聚族索引和非聚族索引

- 聚族索引：索引中data=元数据
  - INNODB中的主键索引存放的就是元数据
- 非聚族索引：索引中data!=元数据，而是元数据的地址
  - 如MyISAM存储引擎的B+树，data存放的是元数据的文件地址

### 覆盖索引

- 一个索引data中有查询字段的值
- 实现方式和存储引擎有关，Momery存储引擎不支持
- 优势
  - 减少IO，B+索引本就有顺序，在IO密集型范围内，读取一次数据IO减少很多
  - 聚族索引支持更好，非聚族索引只做地址缓存严重影响性能

### 细节优化

- 单表6个以内索引
- 一个索引5个字段内
- 根据实际业务优化
- 少用表达式，计算放到业务层
- 优先主键索引，不会触发回表
- 使用前缀索引
- 使用索引排序
- union all、in、or都能使用索引，in最好
- 范围，<,>，但是后面的字段就无法用索引了

### 索引监控

- show status like 'Handler_read%';
- 参数解析
  - Handler_read_first：读取索引第一个条目的次数
  - Handler_read_key：通过index获取数据的次数
  - Handler_read_last：读取索引最后一个条目的次数
  - Handler_read_next：通过索引读取下一条数据的次数
  - Handler_read_prev：通过索引读取上一条数据的次数
  - Handler_read_rnd：从固定位置读取数据的次数
  - Handler_read_rnd_next：从数据节点读取下一条数据的次数

### 案例

##### 预备数据

```sql
SET FOREIGN_KEY_CHECKS=0;
DROP TABLE IF EXISTS `itdragon_order_list`;
CREATE TABLE `itdragon_order_list` (
  `id` bigint(11) NOT NULL AUTO_INCREMENT COMMENT '主键id，默认自增长',
  `transaction_id` varchar(150) DEFAULT NULL COMMENT '交易号',
  `gross` double DEFAULT NULL COMMENT '毛收入(RMB)',
  `net` double DEFAULT NULL COMMENT '净收入(RMB)',
  `stock_id` int(11) DEFAULT NULL COMMENT '发货仓库',
  `order_status` int(11) DEFAULT NULL COMMENT '订单状态',
  `descript` varchar(255) DEFAULT NULL COMMENT '客服备注',
  `finance_descript` varchar(255) DEFAULT NULL COMMENT '财务备注',
  `create_type` varchar(100) DEFAULT NULL COMMENT '创建类型',
  `order_level` int(11) DEFAULT NULL COMMENT '订单级别',
  `input_user` varchar(20) DEFAULT NULL COMMENT '录入人',
  `input_date` varchar(20) DEFAULT NULL COMMENT '录入时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=10003 DEFAULT CHARSET=utf8;

INSERT INTO itdragon_order_list VALUES ('10000', '81X97310V32236260E', '6.6', '6.13', '1', '10', 'ok', 'ok', 'auto', '1', 'itdragon', '2017-08-28 17:01:49');
INSERT INTO itdragon_order_list VALUES ('10001', '61525478BB371361Q', '18.88', '18.79', '1', '10', 'ok', 'ok', 'auto', '1', 'itdragon', '2017-08-18 17:01:50');
INSERT INTO itdragon_order_list VALUES ('10002', '5RT64180WE555861V', '20.18', '20.17', '1', '10', 'ok', 'ok', 'auto', '1', 'itdragon', '2017-09-08 17:01:49');
```

##### 案例一

```sql
select * 
from itdragon_order_list 
where transaction_id = "81X97310V32236260E";

-- 通过查看执行计划发现type=all,需要进行全表扫描
explain 
select * 
from itdragon_order_list 
where transaction_id = "81X97310V32236260E";

-- 优化一、为transaction_id创建唯一索引
create unique index idx_order_transaID on itdragon_order_list (transaction_id);

-- 当创建索引之后，唯一索引对应的type是const，通过索引一次就可以找到结果，普通索引对应的type是ref，表示非唯一性索引赛秒，找到值还要进行扫描，直到将索引文件扫描完为止，显而易见，const的性能要高于ref
 explain 
 select * 
 from itdragon_order_list 
 where transaction_id = "81X97310V32236260E";
 
 -- 优化二、使用覆盖索引，查询的结果变成 transaction_id,当extra出现using index,表示使用了覆盖索引
 explain 
 select transaction_id 
 from itdragon_order_list 
 where transaction_id = "81X97310V32236260E";
```

##### 案例二

```sql
-- 创建复合索引
create index idx_order_levelDate on itdragon_order_list (order_level,input_date);

-- 创建索引之后发现跟没有创建索引一样，都是全表扫描，都是文件排序
explain 
select * 
from itdragon_order_list 
order by order_level,input_date;

-- 可以使用force index强制指定索引
explain 
select * 
from itdragon_order_list 
force index(idx_order_levelDate) 
order by order_level,input_date;

-- 其实给订单排序意义不大，给订单级别添加索引意义也不大
-- 因此可以先确定order_level的值，然后再给input_date排序
explain 
select * 
from itdragon_order_list 
where order_level=3 
order by input_date;
```

## 查询SQL优化

### 查询慢的原因

`网络`、`CPU`、`IO`、`上下文切换`、`系统调用`、`生成统计信息`、`锁等待时间`

### 数据访问优化

- 在无法避免的大量数据过程中，检查**应用程序**和**mysql服务器**是否在检索大量**无需的字段**
- 是请求了无关的字段，如下
  - 查询不需要的字段
  - 多表返回了全部列
  - 总是取出全部列
  - 重复查询相同的数据

### 执行过程优化

![image-20210819211421019](https://i.loli.net/2021/08/19/NMT7RE85UHnIciP.png)

#### =>查询缓存（8.x已经完全删除）

mysql连接器后尝试去命中缓存，命中返回

#### =>解析SQL和预处理：分析器

- 通过关键字将SQL语句进行解析并生成一颗解析树
- mysql解析器将使用mysql语法规则验证和解析查询

#### =>优化SQL执行计划：优化器

##### 初略统计

- 表和索引的页面数、索引个数
- 索引和数据行长度
- 索引分布情况

##### 大多情况下会选择错误的执行计划，原因如下

- 统计信息不准确：**innodb的MVCC会有过个版本**
- 执行计划成本估算不等于实际执行成本：**mysql不知道那些数据实际在内存中和磁盘中**
- 优化器认为的最优的和现实不一样：**基于成本模型，但不是我们认为的最优**
- 不考虑并发执行的查询
- 不考虑不受其控制操作的成本：**比如自定义函数**

##### 优化器策略

- 动态优化：与查询的上下文、取值、索引的函数有关，优化N次
- 静态优化：直接优化解析树，只优化一次

##### 优化器类型

- 重新定义关联顺序

- 外连接转化为内连接

- 使用等价代换规则，可以使用一些等价简化的表达式

- count（）、Min（）、Max（）：索引列不为NULL可以优化这类

- 预估并转化常数表达式，检查到可以是一个常数，即转化为常数

- 覆盖索引扫描，符合‘覆盖索引’条件即用

- 子查询优化：**在某些情况下，子查询会变为缓存**

- 等值传播

  ```sql
  如果两个列的值通过等式关联，那么mysql能够把其中一个列的where条件传递到另一个上：
  explain 
  select film.film_id 
  from film 
  inner join film_actor using(film_id) 
  where film.film_id > 500;
  
  这里使用film_id字段进行等值关联，film_id这个列不仅适用于film表而且适用于film_actor表
  explain 
  select film.film_id 
  from film 
  inner join film_actor using(film_id) 
  where film.film_id > 500 and film_actor.film_id > 500;
  ```

##### 关联查询优化

- 简单关联

  ![image-20210819220411148](https://i.loli.net/2021/08/19/DCAWgGahUbO8vE2.png)

- 有索引关联

  ![](https://i.loli.net/2021/08/19/DCAWgGahUbO8vE2.png)

- 无索引关联

  ![](https://i.loli.net/2021/08/19/DCAWgGahUbO8vE2.png)

  （1）Join Buffer会缓存所有参与查询的列而不是只有Join的列。
  （2）可以通过调整join_buffer_size缓存大小
  （3）join_buffer_size的默认值是256K，join_buffer_size的最大值在MySQL 5.1.22版本前是4G-1，而之后的版本才能在64位操作系统下申请大于4G的Join Buffer空间。
  （4）使用Block Nested-Loop Join算法需要开启优化器管理配置的optimizer_switch的设置block_nested_loop为on，默认为开启。

  **查询optimizer_switch设置情况**：show variables like '%optimizer_switch%';

  ```sql
  -- 案例
  -- 查看不同的顺序执行方式对查询性能的影响：
  explain 
  select 
  film.film_id,
  film.title,
  film.release_year,
  actor.actor_id,
  actor.first_name,
  actor.last_name 
  from film 
  inner join film_actor using(film_id) 
  inner join actor using(actor_id);
  查看执行的成本：
  show status like 'last_query_cost'; 
  
  -- 按照自己预想的规定顺序执行：
  explain 
  select straight_join 
  film.film_id,
  film.title,
  film.release_year,
  actor.actor_id,
  actor.first_name,
  actor.last_name 
  from film 
  inner join film_actor using(film_id) 
  inner join actor using(actor_id);
  -- 查看执行的成本：
  show status like 'last_query_cost'; 
  ```

##### 排序优化

​		无论如何排序都是一个成本很高的操作，所以从性能的角度出发，应该尽可能避免排序或者尽可能避免对大量数据进行排序。推荐使用利用索引进行排序，但是当不能使用索引的时候，mysql就需要自己进行排序，如果数据量小则再内存中进行，如果数据量大就需要使用磁盘，mysql中称之为**filesort**。如果需要排序的数据量小于排序缓冲区(`show variables like '%sort_buffer_size%';`),mysql使用内存进行快速排序操作，如果内存不够排序，那么mysql就会先将树分块，对每个独立的块使用快速排序进行排序，并将各个块的排序结果存放再磁盘上，然后将各个排好序的块进行合并，最后返回排序结果

- 单次排序

  **先读取**查询所需要的所有列，**然后**再根据给定列进行排序，**最后**直接返回排序结果；

  此方式只需要一次顺序IO读取所有的数据，而无须任何的随机IO，问题在于查询的列特别多的时候，会占用大量的存储空间，无法存储大量的数据

- 两次排序

  **第一次**数据读取是将需要排序的字段读取出来，然后进行排序；**第二次**是将排好序的结果按照需要去读取数据行。
  这种方式效率比较低，原因是第二次读取数据的时候因为已经排好序，需要去读取所有记录而此时更多的是随机IO，读取数据成本会比较高
  两次传输的优势，在排序的时候存储尽可能少的数据，让排序缓冲区可以尽可能多的容纳行数来进行排序操作

当需要排序的列的总大小超过**`max_length_for_sort_data`**定义的字节，mysql会选择双次排序，反之使用单次排序，当然，用户可以设置此参数的值来选择排序的方式

### =>执行器

### 特定类型查询优化

#### 优化COUNT()

- MYISAM存储引擎在没有where的条件下count(*)最快
- 近视值，通过EXPLAIN的row取值
- 更复杂优化，考虑覆盖索引扫描或者增加汇总表

#### 优化关联查询

- 确保ON后者using字句中有索引，考虑顺序
- group by和order by中的表达式中涉及一个表中的列相同排序方式才会用到索引

#### 优化子查询

- 尽可能通过关联查询代替

#### 优化Limit查询

- 在很多应用场景中我们需要将数据进行分页，**一般会使用limit加上偏移量的方法实现，同时加上合适的orderby 的子句**，如果这种方式有索引的帮助，效率通常不错，否则的化需要进行大量的文件排序操作，
- 还有一种情况，当偏移量非常大的时候，前面的大部分数据都会被抛弃，这样的代价太高，要优化这种查询的话，**要么是在页面中限制分页的数量，要么优化大偏移量的性能**

#### 优化union查询

- mysql总是通过创建并填充临时表的方式来执行union查询，因此很多优化策略在union查询中都没法很好的使用。
- 经常需要**手工的将where、limit、order by等子句下推到各个子查询中**，**以便优化器可以充分利用这些条件进行优化**

#### 用户自定义变量

- 在查询中混合使用过程化和关系话逻辑的时候，自定义变量会非常有用；用户自定义变量是一个用来存储内容的临时容器，在连接mysql的整个过程中都存在。

- 自定义变量使用

  ```sql
  set @one :=1
  set @min_actor :=(select min(actor_id) from actor)
  set @last_week :=current_date-interval 1 week;
  ```

- 自定义变量的限制

  1. 无法使用查询缓存
  2. 不能在使用常量或者标识符的地方使用自定义变量，例如表名、列名或者limit子句
  3. 用户自定义变量的生命周期是在一个连接中有效，所以不能用它们来做连接间的通信
  4. 不能显式地声明自定义变量地类型
  5. mysql优化器在某些场景下可能会将这些变量优化掉，这可能导致代码不按预想地方式运行
  6. 赋值符号：=的优先级非常低，所以在使用赋值表达式的时候应该明确的使用括号
  7. 使用未定义变量不会产生任何语法错误

- 使用案例

  - 优化排名

    ```sql
    -- 1、在给一个变量赋值的同时使用这个变量
    select actor_id,@rownum:=@rownum+1 as rownum 
    from actor 
    limit 10;
    -- 2、查询获取演过最多电影的前10名演员
    -- 然后根据出演电影次数做一个排名
    select actor_id,count(*) as cnt 
    from film_actor 
    group by actor_id order by cnt desc 
    limit 10;
    ```

  - 避免重新查询更新值

    ```sql
    -- 当需要高效的更新一条记录的时间戳，同时希望查询当前记录中存放的时间戳是什么
    
    -- 分开写法
    update t1 set  lastUpdated=now() where id =1;
    select lastUpdated from t1 where id =1;
    
    -- 自定义变量写法
    update t1 set lastupdated = now() 
    where id = 1 and @now:=now();
    select @now;
    ```

  - 确定取值顺序

    ```sql
    -- 在赋值和读取变量的时候可能是在查询的不同阶段
    
    set @rownum:=0;
    select actor_id,@rownum:=@rownum+1 as cnt 
    from actor where @rownum<=1;
    -- 因为where和select在查询的不同阶段执行，所以看到查询到两条记录，这不符合预期
    
    set @rownum:=0;
    select actor_id,@rownum:=@rownum+1 as cnt 
    from actor 
    where @rownum<=1 order by first_name
    -- 当引入了order by之后，发现打印出了全部结果，
    -- 这是因为order by引入了文件排序，而where条件是在文件排序操作之前取值的  
    
    -- 解决这个问题的关键在于让变量的赋值和取值发生在执行查询的同一阶段：
    set @rownum:=0;
    select actor_id,@rownum as cnt 
    from actor where (@rownum:=@rownum+1)<=1;
    ```
