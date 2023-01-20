---
title: MySQL 数据库优化
date: "2022-01-06T00:00:00Z"
tags: ["基础技术", "database"]
---

MySQL 数据库优化可从以下方面着手：

1. SQL 语句优化
2. 索引优化
3. 锁优化
4. 缓存优化
5. InnoDB 存储引擎优化

## 1. SQL 语句优化

- 检查 select 的 where 条件是否走索引，可用 explain 查看。
- 最小化全表扫描次数。
- 定期使用 `ANALYZE TABLE` 以保持表的统计信息是最新的。

更多，请参考: [https://dev.mysql.com/doc/refman/8.0/en/select-optimization.html](https://dev.mysql.com/doc/refman/8.0/en/select-optimization.html)

## 2. 索引优化

参考: [https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html](https://dev.mysql.com/doc/refman/8.0/en/optimization-indexes.html)

## 3. 锁优化

参考: [https://dev.mysql.com/doc/refman/5.7/en/locking-issues.html](https://dev.mysql.com/doc/refman/5.7/en/locking-issues.html)

## 4. 缓存优化

参考: [https://dev.mysql.com/doc/refman/5.7/en/buffering-caching.html](https://dev.mysql.com/doc/refman/5.7/en/buffering-caching.html)


## 5. InnoDB 存储引擎优化

### 5.1 InnoDB 存储磁盘 I/O 优化

#### 5.1.1 参数 `innodb_buffer_pool_size`

`innodb_buffer_pool_size` 一般调为系统内存的 50%~75%。

#### 5.1.2 参数 `innodb_flush_method`

InnoDB 默认为 `fsync`，但在一些 GNU/Linux 或 Unix 发行版下，fsync 刷新缓存文件至磁盘非常慢，可将  `innodb_flush_method` 调整为 `O_DSYNC`。

#### 5.1.3 参数 `innodb-fsync-threshold`

InnoDB 默认为 `0`，可调整 innodb 缓存阈值，避免单次大文件缓存刷盘。

#### 5.1.4 参数

## 参考文档

1. [MySQL References](https://dev.mysql.com/doc/refman/8.0/en/optimizing-innodb-diskio.html)
