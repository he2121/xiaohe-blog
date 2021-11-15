---

title: "MySQL binlog 设计介绍"
date: 2021-11-10T18:33:24+08:00
tags: ["MySQL", "binlog", "数据库"]
---

## 通过这篇文章你能了解到的知识

- `binlog` 是什么
- `binlog` 的作用
- `binlog` 的三种类型，以及各自优缺点
- `binlog`  文件的结构与内容

## `binlog` 是什么

`Binary Log`，顾名思义是一种二进制格式的日志。具体来说，`binlog` 日志是一组包含了对 **MySQL server** 实例进行数据修改（`update/delete/insert/...`）信息的文件。

## binlog 作用

- 用于复制，如主从复制
- 数据恢复

## binlog 类型

1. 基于**语句** (`Statement-based`)的日志记录 (`SBL`):  事件包含产生数据变化的SQL语句。
1. 基于**行** (`Row-based`) 的日志记录(`RBL`): 描述单个行的变化
2. **混合** (`Mixed`)：上述两者结合使用, 以 `SBL` 为主，特殊情况下切换到 `RBL`

binlog 为什么会有这三种类型？

一开始只有 `statement-based`，但 `statement-based` 存在不少问题，后面才有的 `row` 与 `mixed`。

### `SBL` 优点

- 产生的日志文件少， io 次数少

### `SBL` 缺点

- 在一些**不安全的语句上**，主从复制做不到数据一致，比如

  1. 含有系统函数的语句，可能会在副本上返回不同的值，如 `RAND(), USER(),UUID(), SYSDATE() ....`

  2. 更新一个有 `AUTO_INCREMENT` 列的表

  3. Updates using LIMIT

  4. ...

- 慢 SQL 会在副本中再执行一次


### `RBL` 优点

- 这是最安全的复制形式
- 在 `INSERT/UPDATE/DELETE` 语句中，副本相对主行锁范围更小。

### `RBL 缺点`

- `RBL` 日志文件更大

### 总结

推荐使用 `RBL`，利大于弊，最新 MySQL 版本默认也是使用 `RBL`。

## `binlog` 结构

### binlog 文件结构

`binlog.index` : 文本文件，如下面的例子，列出了当前的二进制日志文件（现在有 6 个）。

`binlog.xxxxxx:` log 二进制文件，

```bash
binlog.000001
binlog.000002
binlog.000003
binlog.000004
binlog.000005
binlog.000006
binlog.index 
```

每个日志文件以一个 4 字节魔数 ( 0xfe b i n) 开头，后面是一组**描述数据修改的事件**，文件格式如下：

```bash
+===================+
| Magic Number	    |
+===================+
| Start Event		|
+===================+
| Event 1			|
+===================+
| ...				|
+===================+
```

一个具体的例子：

```bash
 show BINLOG EVENTS in 'binlog.000001'
+---------------+---------+----------------+-----------+-------------+-------------------------------------------------------------------+
| Log_name      | Pos     | Event_type     | Server_id | End_log_pos | Info                                                              |
+---------------+---------+----------------+-----------+-------------+-------------------------------------------------------------------+
| binlog.000001 | 4       | Format_desc    | 1         | 125         | Server ver: 8.0.26, Binlog ver: 4                                 |
| binlog.000001 | 125     | Previous_gtids | 1         | 156         |                                                                   |
| binlog.000001 | 156     | Anonymous_Gtid | 1         | 233         | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                              |
| binlog.000001 | 233     | Query          | 1         | 337         | use `mysql`; TRUNCATE TABLE time_zone /* xid=3 */                 |
| binlog.000001 | 337     | Anonymous_Gtid | 1         | 414         | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                              |
| binlog.000001 | 414     | Query          | 1         | 523         | use `mysql`; TRUNCATE TABLE time_zone_name /* xid=4 */            |
```

### 事件 Event 常见分类

- **`START_EVENT_V3`**: 每个日志文件的第一个事件， 该事件在MySQL 3.23 至 4.1 中使用，在 MySQL 5.0 中被 `FORMAT_DESCRIPTION_EVENT` 所取代， 从第五个字节开始（魔数后）
- **`QUERY_EVENT`** : `SBL` 的 `binlog` 记录 sql 语句，在 row 模式下记录事务 begin 
- **`STOP_EVENT`** : mysqld 进程停止时记录
- **`ROTATE_EVENT`**: log 文件大小到达设置的最大值，切换到新的 log 文件时写入，切换事件
- **`TABLE_MAP_EVENT`**: 在 binlog 文件是以 ROW 格式记录才会使用，`TABLE_MAP_EVENT` 中记录了表的定义（包括database name, table name，字段定义），记录的每个更改的记录之前都会有一个对应要操作的表的 `TABLE_MAP_EVENT`
- **`WRITE_ROWS_EVENT`**: 插入记录，ROW 格式记录才会使用
- **`UPDATE_ROWS_EVENT`**: 更新记录，ROW 格式记录才会使用
- **`DELETE_ROWS_EVENT`**: 删除记录，ROW 格式记录才会使用
- ...

### Event 结构

`binlog` 版本的演变实质上是 `Event` 的演变

- v1：在MySQL 3.23 中使用

- v3：在MySQL 4.0.2 和4.1中使用

- v4：在MySQL 5.0 及以上版本中使用

v2 已经在 4.0 中使用过，已经废弃了

其中 Event 又分为两个部分: Event header 和 Event data。header 部分的长度由 binlog 版本决定，data 部分的长度又由 header 决定

```bash
+===================+
| event header      |
+===================+
| event data        |
+===================+
```



以下是 Event 的**具体结构**以及**各版本之间的差异**

其中数字代表着这个字段在 Event 中的 limit 与 offset， 例如 0:4，代表着在这个字段的位置在 Event 的 0-4 个字节。

```bash
+=====================================+
| event  | timestamp         0 : 4    |
| header +----------------------------+
|        | type_code         4 : 1    |
|        +----------------------------+
|        | server_id         5 : 4    |
|        +----------------------------+
|        | event_length      9 : 4    | v1 header 共 13 个字节
|        +----------------------------+
|        | next_position    13 : 4    | v3 版本增加。
|        +----------------------------+
|        | flags            17 : 2    | v3 版本增加。 v3 header 共 19 个字节
|        +----------------------------+
|        | extra_headers    19 : x-19 | v4 版本增加。 v4 header 最少 19 个字节
+=====================================+
| event  | fixed part        x : y    |
| data   +----------------------------+
|        | variable part              |
+=====================================+
```

**header 部分各字段解释**

- `timestamp`: 事物开始执行的时间，单位秒
- `type_code`: 事件类型， 见上文 [事件常见分类](#事件 Event 常见分类)，它决定了 data 部分的写入数据
- `server_id`: 来自服务器配置文件中为复制目的设置的 server-id 选项，破除复制中可能的死循环。
- `event_length`:  这个 event 的总长度
- `next_position`: v3 新加，在 v3 版本中代表这个事件开始的位置，在 v4 版本中这个事件结束的位置
- `flags`: v3 新加，一些标志位 https://dev.mysql.com/doc/internals/en/event-flags.html
- `extra_header`: v4 新加，目前是 0，即空

**data 部分**

由事件类型决定，如 v1，v3 query 事件，data 部分构成如下：

```bash
+======================================+
| fixed   | issue_thread      0 : 4    | 触发该语句的线程id
| part    +----------------------------+
|         | timestamp         4 : 4    | 语句执行的时间
|         +----------------------------+
|         | db_name_len       8 : 1    | 使用数据库名长度
|         +----------------------------+
|         | error_code        9 : 2    | 错误码
+======================================+
| variable| db_name                    | db name
| part    +----------------------------+
|         | sql_statment               | sql 语句
+======================================+
```
### 总结

- `binlog` 文件由魔数与事件列表构成，事件列表分成三部分：开始事件 + 数据更新相关的事件 + 切换事件
- 事件的类型决定了事件的格式，因为每种事件所需的信息是不一致的

## 常见问题

1. 主从复制，采取的是 push 模式还是 pull 模式

1. MySQL binlog 为什么不默认使用 Mixed 模式

1. 在使用`RBL`前提下 ，DDL（CREATE/ALTER） 语句怎么记录

1. 使用 `NOW()`的语句是不安全么（`SBL`主从复制  ）
2. 如果让你来设计**开始事件**类型的结构，data 部分需要什么数据

1. `binlog` 与 `redo log` 区别

## 参考

1. https://dev.mysql.com/doc/internals/en/binary-log-overview.html

2. http://mysql.taobao.org/monthly/2018/08/01/

3. https://zhuanlan.zhihu.com/p/33504555

