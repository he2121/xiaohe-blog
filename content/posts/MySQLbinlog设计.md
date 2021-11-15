---
title: "MySQL binlog 设计介绍"
date: 2021-11-10T18:33:24+08:00
tags: ["MySQL", "binlog", "数据库"]
---

## 通过这篇文章你能了解到的知识

- `binlog`是什么
- `binlog` 的作用
- `binlog` 的三种类型，以及各自优缺点
- `binlog`  文件的结构与内容

## `binlog` 是什么

`Binary Log`，顾名思义是一种二进制格式日志。具体来说，`binlog` 日志是一组包含了对 **MySQL server** 实例进行数据修改（`update/delete/insert/...`）信息的文件。

## binlog 作用

- 用于复制，主从复制
- 某些数据恢复操作需要使用 `binlog`

## binlog 类型

1. **基于语句 (`Statement-based`) **的日志记录 (`SBL`):  事件包含产生数据变化的SQL语句。
2. **基于行 (`Row-based`)** 的日志记录(`RBL`): 描述单个行的变化
3. **混合 (`Mixed`)**：上述两者结合使用, 以 `SBL` 为主，特殊情况下切换到 `RBL`

binlog 为什么会有这三种类型？

一开始只有 `statement-based`，但 `statement-based` 存在不少问题，后面才有的 `row-base`与 `mixed`。

### `SBL` 优点

- 产生的日志文件少，减少 io 次数

### `SBL` 缺点

- 在一些**不安全的语句上**，主从复制做不到数据一致，比如

  1. 含有系统函数的语句，可能会在副本上返回不同的值，如 `RAND(), USER(),UUID(), SYSDATE() ....`

  2. 更新一个有 `AUTO_INCREMENT` 列的表

  3. Updates using LIMIT

  4. ...

- 慢 SQL 也会在副本中再执行一次


### `RBL` 优点

- 这是最安全的复制形式
- 在INSERT、UPDATE或DELETE语句中，副本需要更少的行锁。

### `RBL 缺点`

- `RBL` 日志文件更大

### 总结

推荐使用 `RBL`，利大于弊，最新 MySQL 版本默认也是使用 `RBL`。

## `binlog` 结构



## 课后题

1. 主从复制，采取的是 push 模式还是 pull 模式
1. MySQL binlog 为什么不默认使用 Mixed 模式
1. 在使用`RBL`前提下 ，DDL（CREATE/ALTER） 语句怎么记录
2. 使用 `NOW()`的语句是不安全么（`SBL`主从复制  ）
3. `binlog` 与 `redo log` 区别

## 参考

1. https://dev.mysql.com/doc/internals/en/binary-log-overview.html

2. http://mysql.taobao.org/monthly/2018/08/01/

3. https://zhuanlan.zhihu.com/p/33504555

